---
title: 'Lab P2: Full-Stack Observability, SLO Burn Rates, and pgBackRest PITR'
---

# Lab P2: Full-Stack Observability, SLO Burn Rates, and pgBackRest PITR

A dashboard that nobody watches and a database backup that has never been restored are both security theater. The boring default for stateful production operations on NixOS is: **declare a 3-node cluster (`db-01`, `app-02`, `mon-03`) in one flake, stream WAL segments continuously to S3 via `pgBackRest`, alert on multi-window 99.9% SLO error-budget burn rates in Prometheus/Alertmanager, and rehearse a destructive Point-in-Time Recovery (PITR) drill in under 15 minutes**.

## Mental model

```text
+-------------------------------------------------------------------+
|                           NixOS Cluster                           |
|                                                                   |
| +-------------------+   +-------------------+   +---------------+ |
| |   db-01 (.11)     |   |   app-02 (.12)    |   |  mon-03 (.13) | |
| | - PostgreSQL 16   |   | - Web App (:8080) |   | - Prometheus  | |
| | - pgBackRest WAL  |◄──| - Promtail        |──►| - Alertmanager| |
| | - Node Exporter   |   | - Node Exporter   |   | - Loki/Grafana| |
| +---------+---------+   +-------------------+   +---------------+ |
+-----------|-------------------------------------------------------+
            | Continuous encrypted WAL archive + daily full backup
            v
+-------------------------+
| Encrypted Backup Storage |
| (S3 / MinIO Target)     |
+-------------------------+
```

| Node | IP | Role | Recovery Contract |
|---|---|---|---|
| `db-01` | `10.100.80.11` | PostgreSQL 16 + `pgbackrest` continuous WAL archiving | RPO < 60 seconds, RTO < 15 minutes via PITR |
| `app-02` | `10.100.80.12` | HTTP service exposing `http_requests_total{status}` + `promtail` | Stateless; rebuilds from flake in 1 minute |
| `mon-03` | `10.100.80.13` | Prometheus (99.9% SLO burn rules) + Alertmanager + Loki + Grafana | Fires critical alert within 2 minutes of 14.4x error burn |

## Worked examples

### Case 1: Declarative PostgreSQL with `pgBackRest` encrypted S3 archiving (`db-01`)

`pg_dump` once a night loses up to 24 hours of transactions. `pgBackRest` pairs physical base backups with continuous `archive_command` WAL shipping so you can restore to the exact second before an incident.

Save as `hosts/db-01.nix`:

```nix
# hosts/db-01.nix
{ config, pkgs, ... }:
let
  pgbackrestConf = pkgs.writeText "pgbackrest.conf" ''
    [global]
    repo1-type=s3
    repo1-s3-bucket=desk-pgbackrest
    repo1-s3-endpoint=s3.eu-central-1.amazonaws.com
    repo1-s3-region=eu-central-1
    repo1-path=/db-01
    repo1-cipher-type=aes-256-cbc
    repo1-retention-full=7
    process-max=4
    log-level-console=info

    [desk-main]
    pg1-path=/var/lib/postgresql/16
  '';
in
{
  networking.hostName = "db-01";
  system.stateVersion = "26.05";

  environment.systemPackages = [ pkgs.pgbackrest ];
  environment.etc."pgbackrest/pgbackrest.conf".source = pgbackrestConf;

  # S3 credentials & AES-256 passphrase injected via environment variables into systemd
  sops.secrets."pgbackrest-env" = {
    mode = "0400";
    owner = "postgres";
  };

  services.postgresql = {
    enable = true;
    package = pkgs.postgresql_16;
    enableTCPIP = true;
    authentication = ''
      host all all 10.100.80.12/32 scram-sha-256
    '';
    settings = {
      archive_mode = "on";
      archive_command = "${pkgs.pgbackrest}/bin/pgbackrest --stanza=desk-main archive-push %p";
      max_wal_senders = 3;
      wal_level = "replica";
    };
  };

  systemd.services.postgresql.serviceConfig.EnvironmentFile =
    config.sops.secrets."pgbackrest-env".path;

  # Scheduled daily incremental / weekly full backup timer
  systemd.services.pgbackrest-backup = {
    description = "pgBackRest Encrypted Backup to S3";
    serviceConfig = {
      Type = "oneshot";
      User = "postgres";
      EnvironmentFile = config.sops.secrets."pgbackrest-env".path;
      ExecStart = "${pkgs.pgbackrest}/bin/pgbackrest --stanza=desk-main --type=incr backup";
    };
  };

  systemd.timers.pgbackrest-backup = {
    wantedBy = [ "timers.target" ];
    timerConfig = {
      OnCalendar = "daily";
      Persistent = true;
    };
  };

  services.prometheus.exporters.node.enable = true;
  networking.firewall.allowedTCPPorts = [ 5432 9100 ];
}
```

Initialize the stanza and take the initial full baseline backup once:

```bash
sudo -u postgres bash -c '
  set -a && source /run/secrets/pgbackrest-env && set +a
  pgbackrest --stanza=desk-main stanza-create
  pgbackrest --stanza=desk-main --type=full backup
'
```

### Case 2: Prometheus 99.9% SLO recording rules and fast/slow burn-rate alerts (`mon-03`)

A 99.9% availability SLO allows **0.1% error budget** (`0.001`). According to Google SRE multi-window burn-rate alerting:
- **Fast burn (Page):** `14.4x` error budget (`error_rate > 0.0144`) over 5m/1h windows consumes 2% of the 30-day budget in 1 hour.
- **Slow burn (Ticket):** `6x` error budget (`error_rate > 0.006`) over 30m/6h windows.

Save as `hosts/mon-03.nix`:

```nix
# hosts/mon-03.nix
{ config, ... }:
{
  networking.hostName = "mon-03";
  system.stateVersion = "26.05";

  services.prometheus = {
    enable = true;
    port = 9090;
    scrapeConfigs = [
      {
        job_name = "desk-app";
        scrape_interval = "15s";
        static_configs = [ { targets = [ "10.100.80.12:8080" ]; } ];
      }
      {
        job_name = "nodes";
        static_configs = [
          { targets = [ "10.100.80.11:9100" "10.100.80.12:9100" "10.100.80.13:9100" ]; }
        ];
      }
    ];

    alertmanagers = [
      { static_configs = [ { targets = [ "127.0.0.1:9093" ]; } ]; }
    ];

    rules = [
      ''
        groups:
          - name: slo-99.9-availability
            rules:
              - record: job:http_error_ratio:rate5m
                expr: |
                  sum by (job) (rate(http_requests_total{job="desk-app",status=~"5.."}[5m]))
                  /
                  sum by (job) (rate(http_requests_total{job="desk-app"}[5m]))

              - record: job:http_error_ratio:rate1h
                expr: |
                  sum by (job) (rate(http_requests_total{job="desk-app",status=~"5.."}[1h]))
                  /
                  sum by (job) (rate(http_requests_total{job="desk-app"}[1h]))

              - alert: SloFastBurnCritical
                expr: |
                  job:http_error_ratio:rate5m > (14.4 * 0.001)
                  and
                  job:http_error_ratio:rate1h > (14.4 * 0.001)
                for: 1m
                labels:
                  severity: critical
                annotations:
                  summary: "desk-app burning 99.9% SLO budget at >14.4x rate"

              - alert: SloSlowBurnWarning
                expr: job:http_error_ratio:rate1h > (6 * 0.001)
                for: 15m
                labels:
                  severity: warning
                  summary: "desk-app burning 99.9% SLO budget at >6x rate"
      ''
    ];
  };

  services.prometheus.alertmanager = {
    enable = true;
    port = 9093;
    configuration = {
      route = {
        receiver = "ops-webhook";
        group_wait = "10s";
        group_interval = "30s";
      };
      receivers = [
        {
          name = "ops-webhook";
          webhook_configs = [
            { url = "http://127.0.0.1:9999/alerts"; send_resolved = true; }
          ];
        }
      ];
    };
  };

  services.loki = {
    enable = true;
    configuration = {
      auth_enabled = false;
      server.http_listen_port = 3100;
      common = {
        path_prefix = "/var/lib/loki";
        storage.filesystem = {
          chunks_directory = "/var/lib/loki/chunks";
          rules_directory = "/var/lib/loki/rules";
        };
        replication_factor = 1;
        ring.kvstore.store = "inmemory";
      };
      schema_config.configs = [
        {
          from = "2026-01-01";
          store = "tsdb";
          object_store = "filesystem";
          schema = "v13";
          index = { prefix = "index_"; period = "24h"; };
        }
      ];
    };
  };

  services.grafana = {
    enable = true;
    settings.server = {
      http_addr = "10.100.80.13";
      http_port = 3000;
    };
  };

  networking.firewall.allowedTCPPorts = [ 3000 3100 9090 9093 9100 ];
}
```

### Case 3: Disaster simulation — corrupt PostgreSQL data files and burn the SLO budget

1. Record a timestamp marker and insert a canary row on `db-01`, then force a WAL switch so the transaction is archived to S3:

```bash
ssh root@10.100.80.11 '
  sudo -u postgres psql -c "CREATE TABLE IF NOT EXISTS drills (id serial, note text, ts timestamptz default now());"
  sudo -u postgres psql -c "INSERT INTO drills (note) VALUES ('\''pre-corruption-golden-state'\'');"
  RECOVERY_TARGET=$(sudo -u postgres psql -tAc "SELECT now();")
  echo "Target PITR timestamp: $RECOVERY_TARGET"
  sudo -u postgres psql -c "SELECT pg_switch_wal();"
'
```

2. Corrupt the live PostgreSQL heap files on `db-01` using `dd` while traffic hits `app-02`:

```bash
ssh root@10.100.80.11 '
  systemctl stop postgresql
  dd if=/dev/urandom of=/var/lib/postgresql/16/base/1/1259 bs=4k count=10 conv=notrunc
  systemctl start postgresql || true
'
```

`app-02` immediately begins returning HTTP `500` responses as DB queries fail. Within **90–120 seconds**, `job:http_error_ratio:rate5m` crosses `0.0144` (`1.44%`), and Alertmanager delivers `SloFastBurnCritical` to the webhook endpoint.

### Case 4: Verification — Point-in-Time Recovery (PITR) runbook in under 15 minutes

Execute the deterministic PITR recovery runbook on `db-01`:

```bash
# 1. Stop PostgreSQL so no process holds /var/lib/postgresql/16
systemctl stop postgresql

# 2. Restore with pgBackRest using --delta and your target timestamp
sudo -u postgres bash -c '
  set -a && source /run/secrets/pgbackrest-env && set +a
  pgbackrest --stanza=desk-main \
    --delta \
    --type=time \
    "--target=2026-11-20 20:15:00+05:30" \
    --target-action=promote \
    restore
'

# 3. Start PostgreSQL and verify the golden row
systemctl start postgresql
sudo -u postgres psql -c "SELECT * FROM drills;"
```

Output shows `pre-corruption-golden-state`, `app-02` returns `200 OK`, and Alertmanager fires the `resolved` notification—all completed in `< 4 minutes` (well inside the 15-minute RTO target).

## The trap

The trap is **running `pgbackrest restore` without `--delta` (or without cleaning `/var/lib/postgresql/16` first)**, which aborts with `unable to restore to path '/var/lib/postgresql/16' because it contains files`. Passing `--delta` tells `pgBackRest` to checksum existing files on disk and overwrite only damaged or differing blocks, cutting recovery time from 20 minutes to seconds.

The second trap is **testing `archive_command` without loading S3 credentials into the `postgresql.service` environment**. Because PostgreSQL itself spawns `pgbackrest archive-push %p`, the `postgresql` systemd service—not only the `pgbackrest-backup` timer—must carry `EnvironmentFile = config.sops.secrets."pgbackrest-env".path`.

## The boring rule

- Always attach `EnvironmentFile` with S3/cipher secrets to **both** `systemd.services.postgresql` (for continuous `archive-push`) and `systemd.services.pgbackrest-backup` (for scheduled base backups).
- Alert on multi-window error-budget burn rates (`14.4x` fast burn, `6x` slow burn), not raw single-scrape CPU spikes.
- Pass `--delta --type=time --target-action=promote` during PITR drills so recovery finishes in minutes.
- Time every PITR drill with a stopwatch and record the actual RTO in `docs/drills.md`.

## Try this

1. Run `promtool check rules` (from `pkgs.prometheus.cli`) against the extracted YAML rule string in a flake check so syntax errors never merge.
2. Check `sudo -u postgres pgbackrest --stanza=desk-main info` to inspect your backup retention chain and WAL archive range.
3. Compare recovery speed with `--delta` vs wiping `/var/lib/postgresql/16/*` before restore.
