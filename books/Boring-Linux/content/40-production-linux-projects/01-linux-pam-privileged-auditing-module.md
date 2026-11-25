---
title: '01. Project: Custom Linux PAM Module & Privileged Command Auditing Daemon'
---

# Project: Custom Linux PAM Module & Privileged Command Auditing Daemon

Implement a custom **Linux PAM module** (in C or Rust-wrapped) paired with a local **Unix-socket policy daemon** that validates short-lived elevation tokens for `sudo`, logs full process ancestry and TTY session metadata to `journald` / OpenTelemetry, and enforces time-bound least-privilege rules.

## Architecture

```text
sudo <command>
   |
   v  /etc/pam.d/sudo -> pam_ephemeral_token.so
[ Local Policy Daemon (/run/privguard.sock, 0600 root:root) ]
   ├── Validates short-lived signed JWT/OIDC elevation token (TTL <= 15m)
   ├── Inspects /proc/<pid>/status & ppid chain (full process ancestry + TTY)
   ├── Evaluates policy (user, target command, time window, ticket ID)
   └── Emits structured audit event to journald + OTel Collector
```

## Minimal PAM `pam_sm_authenticate` Entry Point (C)

```c
#define PAM_SM_AUTH
#include <security/pam_modules.h>
#include <security/pam_ext.h>
#include <syslog.h>

PAM_EXTERN int pam_sm_authenticate(pam_handle_t *pamh, int flags, int argc, const char **argv) {
    const char *user = NULL;
    if (pam_get_user(pamh, &user, NULL) != PAM_SUCCESS || user == NULL) {
        return PAM_AUTH_ERR;
    }

    // 1. Prompt for short-lived elevation token via pam_prompt()
    // 2. Send {user, tty, pid, token} over /run/privguard.sock with SO_PEERCRED verification
    // 3. Return PAM_SUCCESS only on explicit ALLOW response from policy daemon
    pam_syslog(pamh, LOG_INFO, "privguard evaluated elevation for user=%s", user);
    return PAM_SUCCESS;
}
```

## Acceptance Drills

- Test inside a throwaway container/VM (never directly on your primary host's `/etc/pam.d/sudo` without a root rescue shell open!).
- Verify expired tokens, wrong TTYs, and unreachable policy daemon all **fail closed** (`PAM_AUTH_ERR`).
