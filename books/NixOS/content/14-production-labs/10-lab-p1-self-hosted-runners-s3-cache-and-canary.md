---
title: 'Lab P1: Declarative GitHub Runners, S3 Cache, and Canary Fleet Rollout'
---

# Lab P1: Declarative GitHub Runners, S3 Cache, and Canary Fleet Rollout

Ephemeral cloud runners that download a 4 GB `/nix/store` on every commit are slow, and pushing a merge straight to all production web nodes simultaneously turns a bad config into a full outage. The boring default for production NixOS CI/CD is: **run declarative self-hosted GitHub Actions runners (`services.github-runners`) with a hot local `/nix/store`, gate merges with a `nixosTest` VM suite, push closures to an S3-backed binary cache, and deploy through Colmena in two stages (`@canary` → HTTP health gate → `@prod`)**.

## Mental model

```text
                  +-----------------------------------+
                  |   GitHub Actions CI / Runner Pool |
                  |   (Declarative NixOS Runners)     |
                  +-----------------+-----------------+
                                    | 1. nix flake check (nixosTest)
                                    v 2. nix copy --to s3://desk-nix-cache
                     +------------------------------+
                     |  S3-Backed Binary Cache      |
                     |  (Attic / AWS S3 Store)      |
                     +--------------+---------------+
                                    |
                                    v 3. colmena apply --on @canary
                  +-----------------------------------+
                  |    Colmena Deployment Controller  |
                  +-----------------+-----------------+
                                    |
        +---------------------------+---------------------------+
        | (Stage 1: @canary)        | (Stage 2: @prod)          | (Stage 2: @prod)
        v                           v                           v
+---------------+           +---------------+           +---------------+
| Canary Node   |  HTTP 200 | Prod Node 01  |           | Prod Node 02  |
| (app-canary)  | ────────► | (app-prod-01) |           | (app-prod-02) |
+---------------+  Gate OK  +---------------+           +---------------+
```

| Component | NixOS Mechanism | Why It Matters |
|---|---|---|
| Runner pool | `services.github-runners.<name>` | Ephemeral job directory, persistent `/nix/store`, SOPS-managed registration token |
| Integration test | `pkgs.testers.runNixOSTest` in `checks` | Boots full VM in KVM during PR check to prove the app responds `200 OK` before merge |
| Binary cache | `s3://desk-nix-cache` (`?secret-key=...`) | Native S3 store URI in Nix requires zero database servers if S3/MinIO already exists |
| Canary rollout | Colmena `deployment.tags = [ "canary" ]` | Halts rollout automatically if `app-canary` fails post-activation HTTP checks |

## Worked examples

### Case 1: Declarative GitHub Actions runner pool on NixOS

Configure `runner-01` using `services.github-runners` with `ephemeral = true` so the working tree is wiped after every job while `/nix/store` stays warm across builds.

Save as `hosts/runner-01.nix`:

```nix
# hosts/runner-01.nix
{ config, pkgs, ... }:
{
  networking.hostName = "runner-01";
  system.stateVersion = "26.05";

  sops.secrets."gha-runner-token" = { mode = "0400"; };
  sops.secrets."s3-cache-signing-key" = { mode = "0400"; };

  # Allow the runner service user to use KVM for nixosTest and talk to nix-daemon
  nix.settings.trusted-users = [ "root" "github-runner" ];

  services.github-runners = {
    desk-ci-1 = {
      enable = true;
      url = "https://github.com/desk-org/fleet";
      tokenFile = config.sops.secrets."gha-runner-token".path;
      ephemeral = true;
      replace = true;
      extraLabels = [ "nixos-26.05" "kvm" ];
      extraPackages = with pkgs; [
        git
        colmena
        curl
        jq
        awscli2
      ];
      serviceOverrides = {
        User = "github-runner";
        SupplementaryGroups = [ "kvm" ];
      };
    };
  };

  users.users.github-runner = {
    isSystemUser = true;
    group = "github-runner";
  };
  users.groups.github-runner = { };
}
```

### Case 2: Embedded `nixosTest` VM suite and pre-switch health check

Before any code reaches `@canary`, `nix flake check` boots a VM running the exact `desk-web` module and verifies `/healthz` returns `200 OK`. Furthermore, inside the `desk-web` module itself, add a `systemd` health verification unit so `colmena apply` fails immediately if the daemon crashes on activation.

Save as `modules/desk-web.nix`:

```nix
# modules/desk-web.nix
{ config, lib, pkgs, ... }:
let
  appPkg = pkgs.writers.writePython3Bin "desk-web" { } ''
    from http.server import BaseHTTPRequestHandler, HTTPServer

    class Handler(BaseHTTPRequestHandler):
        def do_GET(self):
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b'{"status":"ok","version":"26.05"}\n')

    HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
  '';
in
{
  systemd.services.desk-web = {
    description = "Desk Web Service";
    wantedBy = [ "multi-user.target" ];
    after = [ "network.target" ];
    serviceConfig = {
      ExecStart = "${appPkg}/bin/desk-web";
      DynamicUser = true;
      Restart = "always";
      # Post-start health gate: systemd marks unit failed if /healthz does not answer 200
      ExecStartPost = "${pkgs.runtimeShell} -c 'for i in 1 2 3 4 5; do ${pkgs.curl}/bin/curl -fsS http://127.0.0.1:8080/healthz && exit 0; sleep 1; done; exit 1'";
    };
  };

  networking.firewall.allowedTCPPorts = [ 8080 ];
}
```

Add the `nixosTest` to `flake.nix`:

```nix
# checks.x86_64-linux.web-vm-test in flake.nix
checks.x86_64-linux.web-vm-test = pkgs.testers.runNixOSTest {
  name = "desk-web-health";
  nodes.web = { ... }: {
    imports = [ ./modules/desk-web.nix ];
  };
  testScript = ''
    web.wait_for_unit("desk-web.service")
    web.wait_for_open_port(8080)
    output = web.succeed("curl -fsS http://127.0.0.1:8080/healthz")
    assert '"status":"ok"' in output
  '';
};
```

### Case 3: Colmena hive with `@canary` and `@prod` tags

Declare the 3-node web cluster in `flake.nix` under `colmena`:

```nix
# colmena output in flake.nix
colmena = {
  meta = {
    nixpkgs = import nixpkgs { system = "x86_64-linux"; };
  };

  defaults = { ... }: {
    imports = [ ./modules/desk-web.nix ];
    system.stateVersion = "26.05";
    nix.settings = {
      substituters = [
        "https://desk-nix-cache.s3.eu-central-1.amazonaws.com"
        "https://cache.nixos.org"
      ];
      trusted-public-keys = [
        "desk-s3-cache-1:9aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789abcdef="
        "cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY="
      ];
    };
  };

  app-canary = {
    deployment.targetHost = "10.100.70.11";
    deployment.tags = [ "canary" ];
  };

  app-prod-01 = {
    deployment.targetHost = "10.100.70.12";
    deployment.tags = [ "prod" ];
  };

  app-prod-02 = {
    deployment.targetHost = "10.100.70.13";
    deployment.tags = [ "prod" ];
  };
};
```

### Case 4: GitHub Actions workflow — test, push S3 cache, canary gate, rolling prod

Save as `.github/workflows/deploy.yml`:

```yaml
# .github/workflows/deploy.yml
name: fleet-rollout
on:
  push:
    branches: [main]

jobs:
  verify-and-rollout:
    runs-on: [self-hosted, nixos-26.05, kvm]
    steps:
      - uses: actions/checkout@v4

      - name: Run flake evaluation & nixosTest VM suite
        run: nix flake check -L

      - name: Build and push system closures to S3 binary cache
        run: |
          nix build .#nixosConfigurations.app-canary.config.system.build.toplevel -o result-canary
          nix copy --to "s3://desk-nix-cache?region=eu-central-1&secret-key=/run/secrets/s3-cache-signing-key" ./result-canary

      - name: Stage 1 — Deploy to @canary
        run: colmena apply --on @canary

      - name: Verify @canary external HTTP health & metrics
        run: |
          for i in $(seq 1 10); do
            curl -fsS http://10.100.70.11:8080/healthz | jq -e '.status == "ok"'
            sleep 1
          done

      - name: Stage 2 — Rolling deploy to @prod (1 node at a time)
        run: colmena apply --on @prod --parallel 1
```

Notice `--parallel 1` on Stage 2: Colmena updates `app-prod-01`, waits for `ExecStartPost` (`curl -fsS http://127.0.0.1:8080/healthz`) to succeed, and only then moves to `app-prod-02`. While a synthetic probe (`while true; do curl -fsS http://lb.desk.internal/healthz; sleep 0.2; done`) runs against the load balancer, zero requests are dropped.

## The trap

The trap is **running `colmena apply --on @prod` with default parallelism (`--parallel 10`) on a two-node production pool**. When both `app-prod-01` and `app-prod-02` restart `desk-web.service` at the exact same millisecond, the load balancer sees zero healthy backends and returns `502 Bad Gateway`. Always pass `--parallel 1` (or configure systemd socket activation) for rolling production updates.

The second trap is **giving the self-hosted GitHub Actions runner unrestricted access to production SSH keys on Pull Requests**. Run `nix flake check` on PRs, but restrict the S3 signing key and Colmena SSH deploy key to a protected `main`-branch job or dedicated deployment runner.

## The boring rule

- Put `SupplementaryGroups = [ "kvm" ]` on `services.github-runners` so `nixosTest` runs at native KVM speed in CI.
- Gate `desk-web.service` activation with `ExecStartPost` checking `/healthz` so a broken binary fails `nixos-rebuild switch` immediately.
- Push closures to S3 (`nix copy --to s3://...`) once from the runner so `@canary` and `@prod` only substitute signed NARs.
- Always roll out `@canary` first, verify external HTTP health, then roll `@prod` with `--parallel 1`.

## Try this

1. Run `nix flake check -L` locally and time the KVM `web-vm-test` execution.
2. Break the JSON returned by `desk-web` (change `"ok"` to `"broken"`), run the canary step against a lab VM, and confirm the pipeline halts at the `@canary` jq assertion while `app-prod-01` and `app-prod-02` remain untouched on the healthy generation.
3. Inspect `nix store ping --store s3://...` against a local MinIO bucket using `?endpoint=http://127.0.0.1:9000&scheme=http`.
