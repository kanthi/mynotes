# Restic backups: a field report and the NixOS mapping

**Goal:** Learn why a rsync-and-snapshot setup is fragile, what Restic gives you, how to handle laptops and containers, and how to express the same design with NixOS.

> [!NOTE]
> Source: Will Richardson, [Backups With Restic](https://willhbr.net/2026/07/19/backups-with-restic/) (19 July 2026). The author says plainly that it is a work in progress. The NixOS snippets are **our own mapping** of his ideas to `services.restic`; they are not from the post. Read [Backups](../07-ops-and-fleet/13-backups.md) first for the book's baseline.

## The problem: backups that look fine

The old setup: machines `rsync` files to a Synology NAS, and `rsnapshot` on the NAS makes daily snapshots.

| Weakness | Consequence |
|----------|-------------|
| No guarantee the `rsync` ran | `rsnapshot` snapshots stale data and the copies mean nothing |
| `rsync` can fail part-way | A snapshot can record a state that never existed |
| Everything in one apartment | One disk of redundancy does not survive a fire or theft |
| Never tested | The author had never restored, so the plan rested on assumptions |

World Backup Day (31 March) is the prompt: check the system works, you can **restore**, and all data is included.

## Why Restic

| Property | Detail |
|----------|--------|
| Portable | Go binary on all major platforms: a Mac backup restores onto Linux and the reverse |
| Chunked, deduplicated | Each run is a snapshot with an ID. Only changed chunks are stored and sent, so changing one byte in a 10 GB file copies one chunk |
| Garbage collection | Chunks referenced by no snapshot can be deleted |
| Many backends | Local disk, SFTP, and cloud storage through rclone |
| Inspect and restore | Browse snapshots, restore one file or everything, or FUSE-mount a snapshot |
| Retention policy | Declare how many restore points to keep and thin older ones |
| `restic copy` | Copy snapshots between repositories, so you back up once to the NAS and copy to the cloud without a second backup run |

Restic deliberately **does not schedule**. That line sits outside the tool, which is where NixOS helps.

## Scheduling: servers and laptops

- **Server:** a timer is enough, because the machine is always on.
- **Laptop:** an hourly schedule can produce no backups at all if the laptop is rarely on at the hour. A fixed schedule that misses its slot gives nothing.

The author wanted **rules**: every few minutes check the time since the last backup, and if it exceeds a few hours, run one. Possible refinements are network state, battery level, and adapting to measured backup speed. He wrote **Rest Up**, a small tool run by a periodic scheduler (cron, launchd or a systemd timer) that keeps state about the last backup and only runs after a configurable wait. It reports status in the shell, tmux status line or SwiftBar, so a failure is seen at the next use.

### NixOS mapping

systemd timers already solve the missed-slot problem with `Persistent=true`: a timer that was missed because the machine was off runs at the next boot or wake.

```nix
{ config, ... }:
{
  sops.secrets.restic-password = { };

  services.restic.backups.home = {
    repository = "sftp:backup@nas.example.invalid:/volume1/restic/laptop";
    passwordFile = config.sops.secrets.restic-password.path;
    paths = [ "/home/alice" ];
    exclude = [ "/home/alice/.cache" ];
    timerConfig = {
      OnCalendar = "hourly";
      Persistent = true;
      RandomizedDelaySec = "10m";
    };
    pruneOpts = [ "--keep-daily 7" "--keep-weekly 5" "--keep-monthly 12" ];
  };
}
```

Our advice: add a check that the **last successful backup is recent** (for example a Prometheus alert on the age of the newest snapshot), since a timer that runs is not proof of a snapshot. See [Observability](../05-services-and-security/10-observability.md). A rules-based "only if older than N hours" wrapper is still useful on laptops, because `Persistent` only catches whole missed runs.

## Containers: back up volumes, not the engine

The author's server keeps all data in Podman volumes. Options:

| Option | Verdict |
|--------|---------|
| Back up `~/.local/share/containers` | Captures gigabytes of images and temporary container state |
| `podman volume export` to tar, piped into Restic | Works: Restic can back up the output of a command |
| **Run Restic in a container with the volumes mounted** | Chosen: Restic sees them as ordinary files, and the host only runs containers |

On NixOS the equivalents are to back up the volume directories directly with `paths`, or to run a short pre-step:

```nix
{
  services.restic.backups.apps = {
    repository = "/srv/backup/restic-apps";
    passwordFile = "/run/secrets/restic-password";
    paths = [ "/var/lib/containers/storage/volumes" ];
    backupPrepareCommand = "systemctl stop myapp.service";
    backupCleanupCommand = "systemctl start myapp.service";
  };
}
```

Stopping or snapshotting a database before copying its files is required for consistency. See [Podman and Docker](../05-services-and-security/09-podman-docker.md) and the database caveats in [A data service pattern](../05-services-and-security/07-data-service.md).

## Replication: 3-2-1 with `restic copy`

- Start with a repository on the NAS, then add an offsite one (the author used Google Drive through rclone with the `drive.file` scope, so rclone only manages files it created).
- Create the second repository with the **same chunker parameters** as the first, so `restic copy` can reuse chunks.
- Use your own cloud API client ID: the shared default rclone client ID effectively did not work.
- A cron script running `restic copy` was serviceable; the author then wrote a small server that does the copies, **prunes**, and exports Prometheus metrics. Clients then only run `restic backup`.

```bash
restic -r sftp:backup@nas.example.invalid:/volume1/restic/laptop init
restic -r rclone:gdrive:restic/laptop init \
  --from-repo sftp:backup@nas.example.invalid:/volume1/restic/laptop \
  --copy-chunker-params
restic -r rclone:gdrive:restic/laptop copy \
  --from-repo sftp:backup@nas.example.invalid:/volume1/restic/laptop
```

## Future direction: a server in the middle

The author wants a combination of his replicator and the **Restic REST server**, which offers an HTTP API and an **append-only mode** so a client can add data but not delete snapshots. That allows backups from less trusted machines (like a VPS) without giving them SFTP access to the NAS, with the server pruning itself and copying offsite.

His measurement: copying about 160 GB over home wifi took roughly **40 GB per hour over SFTP** and **52 GB per hour to the REST server**.

NixOS has a module for the server:

```nix
{
  services.restic.server = {
    enable = true;
    appendOnly = true;
    dataDir = "/srv/restic";
    listenAddress = "127.0.0.1:8000";
    extraFlags = [ "--no-auth" ];   # only behind an authenticating reverse proxy; otherwise use htpasswd
  };
}
```

> [!WARNING]
> Append-only protects against a compromised client deleting history, but pruning then needs a trusted path to the repository. Plan who may prune, and never expose the server without authentication.

## Checklist

- [ ] Backups run from a timer with `Persistent = true`
- [ ] Container state: volumes only, with services quiesced
- [ ] A second repository offsite, same chunker parameters, `restic copy`
- [ ] Alert when the newest snapshot is older than your limit
- [ ] **A restore test on a schedule**, to a different machine
- [ ] Documented setup steps for a new computer (the author's own open item)
