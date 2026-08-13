# Change Requests — Audit Trail

Every configuration change pushed through this system must have a corresponding
Change Request (CR) file in this folder. This provides a full audit trail of
what was changed, who approved it, when it was applied, and which git commit
contains the change.

## Naming convention
CR-XXX-short-description.yml
Example: CR-042-add-vlan100-finance-ho-access.yml

## Status values
- pending   : CR created, not yet applied
- applied   : Successfully pushed to production
- rolled-back : Change was reverted

## Finding what changed on a device
1. Check this folder for the CR that matches the change timeframe
2. Check the git commit hash in the CR file
3. Compare with Oxidized backup repo — device config diff will show the change
