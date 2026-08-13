# Change Playbooks — How to Use

These playbooks are for **ongoing config changes** after the initial hardening baseline
has been pushed. They are separate from the hardening playbooks to keep ongoing
changes auditable and reviewable.

## Workflow for every change

1. Create a feature branch: `git checkout -b change/CR-XXX-description`
2. If the change is device-specific: update `inventory/host_vars/<hostname>/vars.yml`
3. If the change is group-wide: update `inventory/group_vars/<group>/vars.yml`
4. Add a Change Request file: `change_requests/CR-XXX-<description>.yml`
5. Open PR to `staging` → get review → test on dev device via Semaphore
6. Open PR from `staging` to `main` → final approval → production push

## Available change playbooks

| Playbook | Use case | Target scope |
|---|---|---|
| `vlan/add_vlan.yml` | Create new VLAN | Group or specific host |
| `vlan/assign_access_port.yml` | Move port to VLAN | Specific host only |
| `vlan/add_trunk_vlan.yml` | Allow VLAN on uplink | Specific host only |
| `interface/add_ip_interface.yml` | Add routed VLAN interface | Specific host ONLY |
| `interface/add_loopback.yml` | Add loopback interface | Specific host ONLY |
| `routing/add_static_route.yml` | Add static route | Specific host ONLY |

## Rule: NEVER push interface/routing changes to a full group
Interface and routing changes must always target a **specific hostname**.
Never use a group like `huawei_dc_spine` for these — one wrong IP
on a spine or core can take down the entire network.
