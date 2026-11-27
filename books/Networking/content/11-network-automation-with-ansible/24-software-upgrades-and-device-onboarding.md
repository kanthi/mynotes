# Chapter 24 — Software Upgrades and Device Onboarding

## Splitting Upgrades into Two Safe Phases

Never combine image staging and device reload into a single monolithic maintenance window. Split software upgrades into two independent playbooks:

1. **Pre-stage playbook (business hours, zero downtime):** Check free flash space, transfer the signed image via SCP/HTTPS, verify MD5/SHA-512 checksum, and set the boot variable (or expand `.bin` packages on IOS-XE Install Mode).
2. **Cutover playbook (maintenance window, batched with `serial:`):** Capture pre-upgrade routing/interface state, trigger reload, wait for SSH recovery, verify the new running OS version, and run post-upgrade state diff.

## Pre-Stage & Checksum Verification Playbook (IOS-XE)

```yaml
---
- name: Pre-stage IOS-XE firmware image and verify checksum
  hosts: ios_upgrade_wave1
  gather_facts: false
  vars:
    target_version: "17.09.05a"
    target_image: "cat9k_iosxe.17.09.05a.SPA.bin"
    target_md5: "d41d8cd98f00b204e9800998ecf8427e"
    min_free_mb: 1500

  tasks:
    - name: Gather current OS version and flash free space
      cisco.ios.ios_facts:
        gather_subset: [min, hardware]

    - name: Skip hosts already running target version
      ansible.builtin.meta: end_host
      when: ansible_net_version == target_version

    - name: Assert sufficient free space on flash
      ansible.builtin.assert:
        that:
          - (ansible_net_filesystems_info['flash:'].spacefree_kb | int / 1024) >= min_free_mb
        fail_msg: "Insufficient flash space on {{ inventory_hostname }}."

    - name: Copy firmware image via SCP (throttled to protect WAN bandwidth)
      ansible.netcommon.net_put:
        src: "images/{{ target_image }}"
        dest: "flash:/{{ target_image }}"
      throttle: 3
      vars:
        ansible_command_timeout: 1800

    - name: Verify image MD5 hash on device
      cisco.ios.ios_command:
        commands:
          - "verify /md5 flash:{{ target_image }} {{ target_md5 }}"
      register: md5_check
      failed_when: "'Verified' not in md5_check.stdout[0]"
      vars:
        ansible_command_timeout: 300
```

## Day-0 Device Onboarding (ZTP + Ansible)

Zero-Touch Provisioning (ZTP) handles **Day-0** bootstrap (DHCP Option 66/67/150 pulls a minimal bootstrap script that sets management IP, SSH keys, and an automation account). As soon as the device comes online and calls a webhook (or registers in NetBox as `staged`), Ansible takes over **Day-1** provisioning:
1. Upgrades OS to the golden image if factory firmware is older.
2. Applies the full intended configuration rendered from NetBox.
3. Runs end-to-end validation checks and flips NetBox status from `staged` to `active`.
