# Firmware Notes — VRP Version Split Within Huawei S-Series

Source: `Current_Version_All_Switch_FIF.xlsx` (FIF's live asset sheet, Huawei +
Aruba AOS-CX core switches only).

## What the sheet confirmed
- **127 in-scope devices total** (125 Huawei + 2 Aruba AOS-CX 6405v2) —
  matches the SoW device count and the current `inventory/production/hosts`
  count exactly, group by group.
- **`Switch - Admin #1 BSD` is a Brocade ICX 6430-48**, not Huawei — correctly
  absent from `huawei_dc_admin` (4 Huawei admin switches, not 5). This is a
  live example of SoW Exclusion #2 (other-vendor devices out of Phase 1 scope).
- The two `Aruba Instant On 1930` rows share the same serial number
  (`CN39LB31XH`) under two different names (`SW-ACCESS-LT-12` and
  `SW-ACCESS-HO-LT-12 - #1`). Likely a duplicate entry for one physical unit
  rather than two — worth confirming with FIF, doesn't affect automation
  scope either way since both are excluded (no SSH/CLI).

## New finding: two VRP firmware branches inside the same Ansible groups
Most Huawei devices run the **V200** VRP branch (V200R019–V200R024). Eight
devices run the newer **V600** branch (V600R022/V600R024) instead, and they
sit inside groups that are otherwise all-V200:

| Hostname | Model | Group | Firmware |
|---|---|---|---|
| sw-core-a-drc | S8700-4 | huawei_dc_core | V600R022C10SPC500 |
| sw-core-b-drc | S8700-4 | huawei_dc_core | V600R022C10SPC500 |
| sw-access-ho-lt-9-1 | S5735-S48PN4XE-V2 | huawei_ho_access | V600R024C00SPC500 |
| sw-access-ho-lt-9-2 | S5735-S48T4XE-V2 | huawei_ho_access | V600R024C00SPC500 |
| sw-access-ho-lt-9-3 | S5735-S48T4XE-V2 | huawei_ho_access | V600R024C00SPC500 |
| sw-access-ho-lt-9-4 | S5735-S48T4XE-V2 | huawei_ho_access | V600R024C00SPC500 |
| sw-access-ho-lt-9-5 | S5735-S48T4XE-V2 | huawei_ho_access | V600R024C00SPC500 |
| sw-access-ho-lt-17-1 | S5735-S48PN4XE-V2 | huawei_ho_access | V600R024C00SPC500 |

Notably, `huawei_dc_core` is the highest-risk group in the whole push order
(Tier 8, "HIGH RISK" in `playbooks/site.yml`) and its four members are split
2-and-2 across V200 (`sw-core-a-bsd`/`sw-core-b-bsd`, S7703) and V600
(`sw-core-a-drc`/`sw-core-b-drc`, S8700-4) — the two most critical devices in
the fleet are on the branch we have the least CLI confidence in.

V200 and V600 are different VRP major releases; command availability and
syntax for AAA, SNMP, and syslog have changed between them on some platforms.
Rather than assume they're identical, these 8 hosts now carry a
`device_series: huawei_vrp_s_v600` override in their `host_vars`, which routes
them to a new `roles/<role>/tasks/huawei_vrp_s_v600.yml` file (currently a
copy of the V200 file) instead of silently sharing `huawei_vrp_s.yml`.

## What to do with this before go-live
1. When FIF's dev/test devices arrive (SoW Section 4), prioritize getting a
   V600 unit if at all possible, not just any S-series unit — the V200 file
   is already well-covered by the more common S5731 access switches.
2. Run `playbooks/push_ho_access_only.yml` scoped to just
   `sw-access-ho-lt-9-*` first (lower risk) to sanity-check the V600 task
   file before trusting it on `sw-core-a-drc`/`sw-core-b-drc`.
3. If V600 behaves identically to V200 in testing, you can safely delete the
   `huawei_vrp_s_v600.yml` files and the `host_vars` overrides and let those
   8 hosts fall back to the shared `huawei_vrp_s.yml` — the split costs
   nothing to keep and nothing to remove later.
