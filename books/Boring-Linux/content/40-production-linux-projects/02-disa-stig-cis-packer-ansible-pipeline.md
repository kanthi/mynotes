---
title: '02. Project: Automated DISA STIG & CIS Linux Hardening Pipeline'
---

# Project: Automated DISA STIG & CIS Linux Hardening Pipeline

Use **Packer + Ansible** to build hardened RHEL-compatible (**Rocky Linux / AlmaLinux**) golden images with **SELinux enforcing**, `auditd` syscall rules, **FIPS/crypto-policies**, and automated **OpenSCAP** compliance reports published as CI artifacts.

## Golden Image Pipeline

```text
Packer (QEMU / AWS AMI builder)
   ├── Base Rocky/AlmaLinux ISO/Cloud Image
   ├── Ansible Hardening Role (CIS Level 2 / DISA STIG: sysctl, sshd, auditd, fapolicyd, SELinux)
   ├── OpenSCAP Evaluation (oscap xccdf eval -> HTML + ARF XML report)
   └── Fail build if compliance score < 95% or any High-severity finding remains
```

## OpenSCAP Automated Audit Command

```bash
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis \
  --results /tmp/cis-results.xml \
  --report /tmp/cis-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rl9-ds.xml
```

## Acceptance Drills

- Boot the built image and verify `getenforce` returns `Enforcing`, `update-crypto-policies --show` returns `FIPS` (or `FUTURE`), and `auditctl -l` shows active privileged-command and `/etc/passwd` watch rules.
- Publish `cis-report.html` as a GitHub Actions build artifact on every image release.
