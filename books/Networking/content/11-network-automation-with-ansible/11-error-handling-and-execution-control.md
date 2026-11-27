# Chapter 11 — Error Handling and Execution Control

## Controlling Blast Radius in Production

When pushing a BGP policy or switchport change across 40 devices, you never want all 40 failing simultaneously—or a broken primary core router silently continuing onto the secondary core router. Ansible provides fine-grained execution controls to bound failure domains.

## `block`, `rescue`, and `always`

Group related tasks into a `block:` to implement atomic try/catch/finally semantics—ideal for pre-change snapshots, automatic rollback on failed validation, and guaranteed cleanup:

```yaml
- name: Safe configuration rollout with validation and rollback
  block:
    - name: Take pre-change config checkpoint
      cisco.ios.ios_command:
        commands:
          - archive config

    - name: Apply new OSPF timers
      cisco.ios.ios_config:
        src: templates/ospf_timers.j2

    - name: Verify all OSPF neighbors are FULL
      cisco.ios.ios_command:
        commands:
          - show ip ospf neighbor
      register: ospf_check
      failed_when: "'FULL' not in ospf_check.stdout[0]"

  rescue:
    - name: Roll back to pre-change checkpoint on validation failure
      cisco.ios.ios_command:
        commands:
          - configure replace flash:archive-0 force

    - name: Fail the play after rollback so operator is alerted
      ansible.builtin.fail:
        msg: "OSPF validation failed on {{ inventory_hostname }}; config rolled back."

  always:
    - name: Record change outcome to local audit log
      ansible.builtin.lineinfile:
        path: logs/change_audit.log
        line: "{{ ansible_date_time.iso8601 | default('now') }} {{ inventory_hostname }} completed"
        create: true
      delegate_to: localhost
```

## Rolling Batches and Failure Thresholds

| Directive | Scope | Effect |
| :--- | :--- | :--- |
| **`serial: [1, 5, "25%"]`** | Play | Canary rollout: run on 1 host first, then 5, then 25% batches |
| **`max_fail_percentage: 10`** | Play | Abort the entire play if >10% of hosts in the current batch fail |
| **`any_errors_fatal: true`** | Play / Block | Immediately halt the entire play if **any single host** fails |
| **`throttle: 2`** | Task / Block | Cap concurrent executions of a heavy task (e.g., firmware SCP copy) |
| **`failed_when:`** | Task | Define custom failure criteria (e.g., IOS `Invalid input` in stdout) |
| **`changed_when:`** | Task | Override changed status (e.g., `changed_when: false` on read-only commands) |

> [!CAUTION]
> Avoid `ignore_errors: true` in production network playbooks—it hides real outages and lets downstream tasks run against a broken device. Prefer `failed_when:` to handle expected non-fatal output explicitly, or `block`/`rescue` to recover cleanly.

## Delegation and `run_once`

Because network devices cannot run local scripts, you frequently combine two directives:
- **`delegate_to: localhost`**: Runs a task on the Ansible control node (e.g., querying NetBox API, writing a CSV report, or sending a Slack webhook).
- **`run_once: true`**: Executes the task once per play instead of once per inventory host (e.g., creating a timestamped backup directory before 50 switches write to it).

```yaml
- name: Create dated backup directory on control node
  ansible.builtin.file:
    path: "backups/{{ lookup('pipe', 'date +%Y-%m-%d') }}"
    state: directory
    mode: "0750"
  delegate_to: localhost
  run_once: true
```
