# FIF Network Automation — Ansible Repository

## Overview
This repository contains Ansible playbooks and roles for PT Federal International Finance (FIF)
network device automation, covering configuration hardening push for 127 Huawei and Aruba devices
across DC Switch, DRC Switch, and HO Switch sites (per SoW v2.0), plus an in-office lab overlay
used to rehearse the CI/CD pipeline before touching production.

## Repository Structure
```
fif-network-automation/
├── inventory/
│   ├── production/         # Real FIF devices — Huawei + Aruba
│   │   ├── hosts           # Static inventory — replace FILL_IP with actual IPs
│   │   ├── group_vars/      # Per-group variables (device_series drives role dispatch)
│   │   └── host_vars/       # Per-device overrides (VLANs, IPs, routes — see below)
│   └── lab/                 # Office mock-up — Cumulus Linux + Brocade CES/ICX
│       ├── hosts
│       ├── group_vars/
│       └── host_vars/
├── roles/                   # One role per config concern, e.g. roles/snmp/
│   └── <role>/tasks/
│       ├── main.yml         # Dispatcher — picks the file below by device_series
│       ├── huawei_vrp_s.yml
│       ├── huawei_vrp_s_v600.yml   # S-series units on VRP V600 — see docs/firmware-notes.md
│       ├── huawei_vrp_ce.yml
│       ├── aruba_aoscx.yml
│       ├── cumulus_linux.yml   # lab only
│       └── brocade_icx.yml     # lab only
├── playbooks/
│   ├── site.yml              # Full production push, correct tier order
│   ├── push_dc_only.yml
│   ├── push_ho_access_only.yml
│   ├── dry_run.yml
│   ├── lab_pipeline_test.yml # Runs the SAME roles against the lab inventory
│   └── changes/               # Day-2 change playbooks (VLAN, routing, interfaces)
├── change_requests/          # CR audit trail — see change_requests/README.md
├── oxidized/
│   ├── router.db             # Production device list
│   └── router_lab.db         # Lab device list — keep backups in a separate repo/folder
├── docs/
│   ├── lab-strategy.md        # What the lab does / does not prove
│   └── firmware-notes.md      # VRP V200 vs V600 split found in FIF's asset sheet
└── .github/
    ├── CODEOWNERS
    └── workflows/ci.yml      # yamllint + ansible-lint + syntax-check on every PR
```

## Why roles are split by `device_series`, not just by vendor collection
The original design branched only on `ansible_network_os` (Huawei collection vs. Aruba
collection), which silently lumped Huawei **S-Series** (campus/access, e.g. S5731, S7703, S8700)
and **CE-Series** (CloudEngine datacenter, e.g. CE6857, CE6881, CE8861) into one code path.
The SoW (Section 3.1) explicitly calls for **separate task files** for these two families because
their CLI differs in places. Every role now has:
```yaml
device_series: huawei_vrp_s | huawei_vrp_ce | aruba_aoscx | cumulus_linux | brocade_icx
```
set in `group_vars`, and `roles/<role>/tasks/main.yml` `include_tasks`'s the matching file. This
also makes adding a future platform (e.g. a new vendor) a matter of dropping in one file per role,
not editing every task with more `when:` conditions.

## Branch Strategy
| Branch   | Purpose                        | Who can merge      |
|----------|--------------------------------|--------------------|
| `dev`    | Active development and testing | Engineers          |
| `staging`| Pre-production validation      | Senior engineer    |
| `main`   | Production — triggers Semaphore webhook | Approver  |

Direct push to `main` and `staging` is blocked. All changes go through PR review, and
`.github/workflows/ci.yml` must pass (lint + syntax-check) before merge.

## Push Order (CRITICAL — do not change)
1. `huawei_ho_access` — 65 HO floor access switches (lowest risk)
2. `huawei_ho_mgmt` — HO management switch
3. `huawei_ho_serverfarm` — HO server farm switch
4. `huawei_dc_admin` — DC/DRC admin switches
5. `huawei_dc_tor` — 41 DC/DRC Top-of-Rack switches
6. `huawei_dc_leaf` — DC/DRC leaf switches
7. `huawei_dc_spine` — DC/DRC spine switches (HIGH RISK)
8. `huawei_dc_core` — DC/DRC core switches (HIGH RISK)
9. `aruba_ho_core` — Aruba 6405v2 HO core (PUSH LAST — most critical)

## Handling future attribute changes (SNMP key, IP, syslog, VLAN, routing...)
- **Fleet-wide values** (syslog server, NTP servers, SNMP community) live in
  `inventory/production/group_vars/all/vars.yml` and `all/vault.yml`. Change once, PR, re-run `site.yml`.
- **Per-device values** (VLAN SVIs, loopbacks, static routes) live in
  `inventory/production/host_vars/<hostname>/vars.yml` and are picked up automatically by the
  `ip_interface`, `loopback`, and `static_route` roles — no playbook edits needed.
- **New VLAN / port / route requests** go through `playbooks/changes/*` and a
  `change_requests/CR-XXX-*.yml` record — see `change_requests/README.md`.
- **A brand-new site or vendor** only needs a new inventory group + group_vars +
  (if it's a new vendor) one new task file per role — the dispatcher pattern above.

## Before You Start
1. Replace all `FILL_IP` in `inventory/production/hosts` with actual management IPs from FIF
2. Replace all `FILL_IP` in `oxidized/router.db` with actual management IPs
3. Set your Ansible Vault password in Semaphore Key Store
4. Confirm with FIF: syslog destination IP (Elastic endpoint), SNMP community strings, NTP servers
5. For the lab rehearsal: fill in `inventory/lab/hosts` and run `playbooks/lab_pipeline_test.yml`
   (see `docs/lab-strategy.md` for what it does and doesn't prove)

## Excluded Devices
- Aruba Instant On 1930 (2 units, HO Lt.12) — cloud-managed, no SSH/CLI access
