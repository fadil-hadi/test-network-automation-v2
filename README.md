# Testing Network Automation — Ansible Repository

## Overview
This repository contains Ansible playbooks and roles for testing network device automation, covering
configuration hardening push for in-office lab overlay used to rehearse the CI/CD pipeline
before touching production.

## Repository Structure
```
test-network-automation/
├── inventory/
│   └── lab/                 # Office mock-up
│       ├── hosts
│       ├── group_vars/
│       └── host_vars/
├── roles/                   # One role per config concern, e.g. roles/snmp/
│   └── <role>/tasks/
├── playbooks/
│   └── lab_pipeline_test.yml # Runs the SAME roles against the lab inventory
├── change_requests/          # CR audit trail — see change_requests/README.md
├── oxidized/
│   ├── router.db             # Production device list
│   └── router_lab.db         # Lab device list — keep backups in a separate repo/folder
└── .github/
    ├── CODEOWNERS
    └── workflows/ci.yml      # yamllint + ansible-lint + syntax-check on every PR
```


## Branch Strategy
| Branch   | Purpose                        | Who can merge      |
|----------|--------------------------------|--------------------|
| `dev`    | Active development and testing | Engineers          |
| `staging`| Pre-production validation      | Senior engineer    |
| `main`   | Production — triggers Semaphore webhook | Approver  |

Direct push to `main` and `staging` is blocked. All changes go through PR review, and
`.github/workflows/ci.yml` must pass (lint + syntax-check) before merge.
