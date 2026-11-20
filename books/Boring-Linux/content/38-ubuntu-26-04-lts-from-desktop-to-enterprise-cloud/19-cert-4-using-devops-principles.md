---
title: 'Chapter 19: Certification 4 — Using DevOps Principles'
---

# Chapter 19: Certification 4 — Using DevOps Principles

**Target Profile:** Infrastructure-as-Code (IaC), fleet lifecycle management, container/VM virtualization, and deep performance/network troubleshooting.  
**Duration:** 90–120 Minutes | **Format:** Performance Labs + Scenario & Knowledge Items

| Module | Topic Domain | Exam Weight |
|---|---|---|
| **Module 4.1** | Understanding Deployment Technologies | **34%** |
| **Module 4.2** | Monitoring & Troubleshooting Systems | **34%** |
| **Module 4.3** | Using Tools for Automation and System Updating | **32%** |

---

## Module 4.1: Understanding Deployment Technologies (Weight: 34%)

### 4.1.1 Containerized Workloads (Docker & LXD)

- **Docker & Docker Compose Operations:**
  - **Container Lifecycle:** Running, inspecting, and debugging application containers (`docker run -d --name api --restart unless-stopped -p 127.0.0.1:8080:8080 -e APP_ENV=prod -v api_data:/var/lib/api myorg/api:1.0`, `docker ps -a`, `docker logs --tail 50 -f api`, `docker exec -it api sh`, `docker inspect`, `docker stats --no-stream`, `docker stop`, `docker rm`).
  - **Image Authoring (`Dockerfile`):** Multi-stage builds, minimizing layer size, running as non-root `USER`, `COPY --chown`, `HEALTHCHECK`, and building with `docker build -t myorg/api:1.0 .`.
  - **Networks, Volumes & Compose:** Creating isolated bridge networks (`docker network create backend-net`), persistent named volumes (`docker volume create pg_data`), and declarative multi-service stacks in `compose.yaml` (`docker compose up -d`, `docker compose ps`, `docker compose logs`, `docker compose down`).
- **LXD System Containers & Virtual Machines:**
  - **Initialization & Lifecycle:** Initializing storage pools and bridges (`sudo lxd init --auto`), launching full Ubuntu system containers (`lxc launch ubuntu:26.04 web-ct`) or QEMU virtual machines (`lxc launch ubuntu:26.04 db-vm --vm`), executing commands (`lxc exec web-ct -- apt update`), pushing/pulling files (`lxc file push app.conf web-ct/etc/app.conf`).
  - **Profiles, Resource Limits, Snapshots & Proxy Devices:** Applying CPU/memory constraints (`lxc config set web-ct limits.cpu=2 limits.memory=2GiB`), managing reusable profiles (`lxc profile show default`, `lxc profile create prod`), taking stateful snapshots (`lxc snapshot web-ct pre-upgrade`, `lxc restore web-ct pre-upgrade`), and exposing host ports to containers (`lxc config device add web-ct http80 proxy listen=tcp:0.0.0.0:8080 connect=tcp:127.0.0.1:80`).

### 4.1.2 Virtualization Management (`libvirt` / `virsh`)

- **KVM/QEMU Provisioning via `virsh` & `virt-install`:**
  - Auditing hardware virtualization extensions (`kvm-ok`, `lsmod | grep kvm`).
  - Provisioning headless guests with `virt-install` (`--name srv-vm --memory 2048 --vcpus 2 --disk size=20 --os-variant ubuntu24.04 --network bridge=virbr0 --graphics none --console pty,target_type=serial`).
  - Managing guest lifecycle via `virsh`: listing domains (`virsh list --all`), starting/stopping (`virsh start`, `virsh shutdown`, `virsh reboot`, `virsh destroy` hard power-off), enabling boot autostart (`virsh autostart <vm>`), inspecting resources (`virsh dominfo`, `virsh domifaddr`, `virsh domblklist`), attaching serial consoles (`virsh console <vm>`), and exporting/defining XML domain definitions (`virsh dumpxml <vm> > vm.xml`, `virsh define vm.xml`, `virsh undefine <vm>`).

---

## Module 4.2: Monitoring & Troubleshooting Systems (Weight: 34%)

### 4.2.1 Compute, Memory & Storage Performance Analysis

| Subsystem | Diagnostic Toolset | Key Metrics & Bottleneck Signatures |
|---|---|---|
| **CPU & Scheduling** | `uptime`, `vmstat 1 5`, `mpstat -P ALL 1`, `pidstat -u 1`, `top`/`htop` | `r` (run queue) > CPU core count indicates CPU saturation; high `sy` + `cs` indicates excessive context switching; high `wa` indicates threads blocked waiting on disk/NFS I/O. |
| **Memory & OOM** | `free -m`, `/proc/meminfo`, `vmstat 1` (`si`/`so`), `dmesg -T \| grep -iE 'oom\|killed'` | Notice distinction between `free` (unused RAM) and `available` (reclaimable page cache + free). Sustained non-zero `si`/`so` (swap-in/swap-out) indicates swap thrashing; kernel OOM events record victim PID and `oom_score_adj` in `dmesg`/journal. |
| **Disk I/O & Inodes** | `iostat -xz 1`, `iotop -oP`, `df -h`, `df -i`, `lsof +L1` | `%util` near 100% with rising `await` (ms) and `aqu-sz` (queue depth) shows block saturation; `df -i` at 100% `IUse%` blocks file creation even when GBs exist; `lsof +L1` finds deleted open files still holding disk blocks until the daemon restarts. |

- **Real-Time Degradation Response & `systemd` Resource Control:**
  - Tuning kernel runtime parameters with `sysctl` (`sudo sysctl -w vm.swappiness=10`, persisting in `/etc/sysctl.d/99-tuning.conf` and reloading with `sudo sysctl --system`).
  - Enforcing live cgroups v2 memory and CPU boundaries on noisy services (`sudo systemctl set-property noisy-app.service CPUQuota=150% MemoryMax=1G`).

```bash
# Rapid 4-command triage for a slow server:
uptime && vmstat 1 3
iostat -xz 1 2
df -hT && df -i
sudo lsof +L1 2>/dev/null
```

### 4.2.2 Deep Network Diagnostics (`iproute2`, `ss`, `dig`, `tcpdump`)

- **Modern Networking Stack (`iproute2`):** Inspecting interface state and dropped packet counters (`ip -s link show`), IPv4/IPv6 addresses (`ip -br addr show`), routing tables (`ip route show`, `ip route get 10.20.30.40` to verify exact source IP and egress interface), and policy routing rules (`ip rule show`).
- **Socket & Port Analysis (`ss`):** Identifying which process owns a listening port (`sudo ss -tulnp`), summarizing socket memory and states (`ss -s`), and filtering established or `TIME-WAIT` TCP sessions (`ss -tan state established`, `ss -tan '( dport = :443 or sport = :443 )'`).
- **DNS Troubleshooting (`dig` & `resolvectl`):** Querying specific resolvers (`dig @1.1.1.1 example.com A +noall +answer`), tracing full delegation from root servers (`dig +trace example.com`), checking reverse PTR (`dig -x 192.0.2.10`), and inspecting `systemd-resolved` per-link DNS routing and cache (`resolvectl status`, `sudo resolvectl flush-caches`).
- **Packet Forensics (`tcpdump`):** Capturing live traffic without DNS/port name resolution (`-nn`) to verify whether packets arrive on the wire, whether TCP `SYN` receives `SYN-ACK` or `RST`, or whether a firewall silently drops traffic:

```bash
# Capture TCP port 443 traffic to/from 10.10.20.15, write to a pcap file, and inspect
sudo tcpdump -i eth0 -nn -vv 'tcp port 443 and host 10.10.20.15' -c 50 -w /tmp/tls-debug.pcap
sudo tcpdump -nn -r /tmp/tls-debug.pcap

# Diagnose port conflict: find exact PID and systemd unit bound to TCP 8080
sudo ss -tulnp 'sport = :8080'
```

### 4.2.3 Automated Monitoring & Alerting

- **`systemd` Failure Hooks (`OnFailure=`):** Triggering an automated alert or remediation unit whenever a critical service enters the `failed` state (`OnFailure=notify-ops@%n.service` inside `[Unit]`).
- **Proactive Endpoint & Resource Probes:** Combining `curl -fsS --max-time 5` HTTP status checks, disk-usage threshold checks, and Prometheus `node_exporter` metrics.

---

## Module 4.3: Using Tools for Automation and System Updating (Weight: 32%)

### 4.3.1 Bash & Python Automation with Git Version Control

- **Production Bash Scripting:** Writing defensive, idempotent scripts (`#!/usr/bin/env bash`, `set -euo pipefail`), using `trap 'rm -f "$TMPFILE"' EXIT` cleanup handlers, validating argument counts (`$#`), quoting variables (`"$var"`), and returning meaningful exit codes.
- **Python System Automation:** Writing structured administration scripts using standard library modules (`subprocess.run([...], check=True, capture_output=True, text=True)`, `pathlib.Path`, `os`, `shutil`, `json`, `urllib.request`) and `yaml` / `requests`.
- **Git Workflows:** Repository lifecycle (`git init`, `git clone`), staging and committing (`git add -p`, `git commit -m`), inspecting history and diffs (`git status`, `git log --oneline --graph`, `git diff --staged`), branching and merging (`git switch -c feature/fix`, `git merge`, resolving conflict markers `<<<<<<<` / `=======` / `>>>>>>>`), and remote synchronization (`git fetch`, `git pull --rebase`, `git push`).

```bash
#!/usr/bin/env bash
# /usr/local/sbin/check-service-health.sh — defensive production health check
set -euo pipefail

SERVICE="${1:-nginx.service}"
ENDPOINT="${2:-http://127.0.0.1/healthz}"
TMP_OUT="$(mktemp)"
trap 'rm -f "$TMP_OUT"' EXIT

if ! systemctl is-active --quiet "$SERVICE"; then
  logger -p daemon.err "Health check failed: $SERVICE is not active; restarting"
  systemctl restart "$SERVICE"
fi

http_code="$(curl -sS -o "$TMP_OUT" -w '%{http_code}' --max-time 5 "$ENDPOINT")"
if [[ "$http_code" != "200" ]]; then
  logger -p daemon.err "Endpoint $ENDPOINT returned HTTP $http_code"
  exit 1
fi
```

### 4.3.2 Configuration Management & Provisioning (`Ansible` & `cloud-init`)

- **Ansible Architecture:** Structuring static INI or YAML inventories (`inventory.yml`) with host groups and variables, testing connectivity (`ansible all -i inventory.yml -m ping`), and authoring idempotent Playbooks using core modules (`ansible.builtin.apt`, `ansible.builtin.copy`, `ansible.builtin.template` with Jinja2 `.j2`, `ansible.builtin.systemd_service`, `ansible.builtin.user`, `ansible.builtin.file`, `ansible.builtin.lineinfile`, `community.general.ufw`), `handlers` (`notify: Restart Nginx`), `when` conditionals, and pre-flight validation (`ansible-playbook --syntax-check`, `ansible-playbook --check --diff`).

```yaml
# site.yml — Idempotent Ansible Playbook for Ubuntu Web Fleet
---
- name: Harden and configure Ubuntu web nodes
  hosts: webservers
  become: true
  vars:
    app_port: 8080
  tasks:
    - name: Ensure Nginx and UFW are installed
      ansible.builtin.apt:
        name:
          - nginx
          - ufw
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: Deploy Nginx reverse-proxy virtual host from Jinja2 template
      ansible.builtin.template:
        src: templates/app-vhost.conf.j2
        dest: /etc/nginx/sites-available/app.conf
        owner: root
        group: root
        mode: "0644"
        validate: /usr/sbin/nginx -t -c /etc/nginx/nginx.conf
      notify: Reload Nginx

    - name: Enable Nginx site symlink
      ansible.builtin.file:
        src: /etc/nginx/sites-available/app.conf
        dest: /etc/nginx/sites-enabled/app.conf
        state: link
      notify: Reload Nginx

    - name: Ensure Nginx service is enabled and running
      ansible.builtin.systemd_service:
        name: nginx
        enabled: true
        state: started

  handlers:
    - name: Reload Nginx
      ansible.builtin.systemd_service:
        name: nginx
        state: reloaded
```

### 4.3.3 Enterprise Fleet & Patch Management (Ubuntu Pro, ESM, Livepatch, Landscape)

- **Ubuntu Pro CLI (`pro`):**
  - Attaching hosts to an Ubuntu Pro subscription (`sudo pro attach <TOKEN>`), inspecting active entitlements (`pro status`), enabling/disabling services (`sudo pro enable esm-infra esm-apps livepatch usg`), and auditing package CVE exposure across `Main` and `Universe` (`pro security-status`, `pro fix CVE-YYYY-NNNN`).
- **Canonical Livepatch:**
  - Applying high and critical kernel CVE patches in memory without rebooting (`sudo canonical-livepatch status --verbose`, `sudo canonical-livepatch refresh`).
- **Canonical Landscape:**
  - Enrolling fleet nodes (`sudo apt install landscape-client && sudo landscape-config --account-name ... --computer-title ...`), grouping servers by tag/profile, scheduling staged package update waves, auditing USN compliance, and executing remote management scripts across thousands of Ubuntu nodes.

---

## Certification 4 Pre-Submission Verification Checklist

- [ ] **Docker, LXD & KVM:** Running containers have `--restart unless-stopped` (or `restart: unless-stopped` in `compose.yaml`); LXD containers/VMs show `RUNNING` in `lxc list`; libvirt guests required at boot have `Autostart: enable` in `virsh dominfo <vm>`.
- [ ] **Performance & Network Triage:** Any `sysctl` parameter tuned via `sysctl -w` is also written to `/etc/sysctl.d/*.conf` and verified with `sudo sysctl --system`; deleted open files holding disk space (`lsof +L1`) have had their owning service restarted.
- [ ] **Bash, Git & Ansible:** Custom scripts have `chmod +x` and pass `bash -n <script>`; Git commits are clean (`git status` reports nothing untracked/unstaged); Ansible playbooks pass `ansible-playbook -i <inventory> <playbook.yml> --check` with `changed=0` on a second idempotent run.
- [ ] **Ubuntu Pro & Livepatch:** `pro status` shows the requested entitlements (`esm-infra`, `esm-apps`, `livepatch`) as `enabled`.
