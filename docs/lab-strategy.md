# Lab / Mock-up Strategy — Cumulus Linux + Brocade CES/ICX

## What this lab proves
The office mock-up (Cumulus Linux switches + Brocade CES/ICX switches) validates
the **automation pipeline**, not the **vendor CLI**:

- Git branch protection (`dev` → `staging` → `main`) and PR review gates
- Ansible Vault secret handling end-to-end via Semaphore's Key Store
- Semaphore job templates, task chaining, and push-order enforcement
- Oxidized: SSH polling, diffing, committing to the backup Git repo,
  and the "current config + full history + diff" viewer experience
- The `roles/*/tasks/main.yml` dispatcher pattern itself (does the right
  task file get included for the right `device_series`?)
- CI checks in `.github/workflows/ci.yml` actually gating a PR

## What this lab does NOT prove
- Huawei VRP S-series or CE-series exact CLI syntax
- Aruba AOS-CX exact CLI/REST behavior
- Any FastIron-vs-VRP-vs-AOS-CX quirks (prompt handling, timing, paging,
  command availability per firmware version)

Cumulus Linux (Debian-based, NCLU/systemd) and Brocade FastIron (ICX/CES)
are **different vendors with different CLI families** than the production
target (Huawei VRP + Aruba AOS-CX). They share the *shape* of the roles
(credentials, hardening, syslog, snmp, ntp, vlan, ip_interface, etc.) but
each has its own `roles/<role>/tasks/<device_series>.yml` file — see the
dispatcher pattern in each role.

## Why this still matters
1. It lets you rehearse the entire CI/CD flow — including failure modes
   (a bad PR, a Vault secret typo, a broken Semaphore job template) —
   without any risk to FIF's production network.
2. It exercises the *generic* parts of the design (inventory structure,
   group_vars/host_vars precedence, change-request workflow, Oxidized
   Git-as-viewer pattern) which are vendor-agnostic.
3. It gives you a rehearsed, repeatable demo you can show FIF before
   the real UAT window.

## What still requires real hardware
Per the SoW (Section 4, "Required Components" and Section 6, "Acceptance
Criteria"), FIF is contractually required to provide **at minimum 1
Huawei S-series device and 1 Aruba AOS-CX device** for dev/test, and
acceptance requires a successful push to **at minimum 5 test devices**
on the real platforms. The Cumulus/Brocade lab is a *rehearsal*, not a
substitute for that requirement — plan to get the real dev/test units
from FIF as early as possible so S-series vs CE-series CLI divergences
(if any) surface before UAT, not during it.

## Running the lab pipeline
```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory/lab/hosts playbooks/lab_pipeline_test.yml --check   # dry run
ansible-playbook -i inventory/lab/hosts playbooks/lab_pipeline_test.yml          # apply
```
Point a second Oxidized instance (or a second config file) at
`oxidized/router_lab.db` so lab backups land in a separate repo/folder
and never mix with the production backup history.
