# Chapter 34 — Building an Automation Practice

## Why Technology Is Only 30% of the Problem

Most stalled network automation initiatives do not fail because Ansible or NetBox lacked a feature. They fail because one enthusiastic engineer built a brilliant repository in isolation, nobody else on the team trusted or understood it, and everyone reverted to manual SSH during the first high-pressure incident.

## Winning Team Adoption

1. **Solve real pain first, not shiny tech:** Automate the tedious tasks every engineer hates—nightly backups, VLAN provisioning across 20 switches, pre-upgrade flash checks, and P1 triage bundles—before pitching full zero-touch CI/CD.
2. **Make the automated path easier than manual SSH:** If running a playbook takes 20 minutes of fighting Python virtualenvs while SSH takes 2 minutes, engineers will SSH. Wrap common tasks in a `Makefile`, container EE, or AWX web survey.
3. **Pair-program and review together:** Treat pull request reviews as collaborative design sessions, not gatekeeping exams. Every engineer who reviews a Jinja2 template learns how the system works.
4. **Document runbooks inside the repo:** Keep a crisp `README.md` with exact copy-pasteable commands for onboarding, dry runs, rollbacks, and emergency break-glass procedures.

## Measuring What Actually Matters

Track four concrete operational metrics to demonstrate value to leadership and guide where to invest next:

| Metric | How to Measure | Target Trend |
| :--- | :--- | :--- |
| **Change failure rate** | % of network changes causing rollbacks or P1/P2 incidents | Downward toward <2% |
| **Configuration drift score** | % of fleet passing nightly golden-config compliance checks | Upward toward >98% |
| **Mean Time to Deploy (MTTD)** | Time from approved ticket to verified production completion | Hours/days → minutes |
| **Automation coverage** | % of recurring operational changes executed via Git/AWX instead of manual CLI | Steady quarterly growth |

## Sustaining the Practice Over Years

- **Budget 15–20% maintenance time:** Pin and upgrade `ansible-core`, vendor collections, and Execution Environments on a predictable quarterly cadence.
- **Retire manual habits deliberately:** Once an automated workflow is proven reliable for a domain (e.g., NTP, DNS, campus VLANs), revoke routine manual write habits and route all updates through Git so drift never creeps back in.
- **Invest in shared skills:** Rotate ownership of roles and CI pipelines across the team so no single engineer becomes a bottleneck or single point of failure.
