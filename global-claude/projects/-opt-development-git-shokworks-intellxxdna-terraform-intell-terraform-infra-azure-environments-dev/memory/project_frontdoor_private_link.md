---
name: Front Door Private Link experiment
description: Attempted to add Azure Front Door Premium Private Link to Container App Environment in dev; failed and rolled back
type: project
---

Attempted to configure Front Door Premium Private Link so origins route through Microsoft's private backbone (not public internet).

**What was done:**
- Upgraded AzureRM provider to 4.x (needed for `managedEnvironments` target_type)
- Set Container App Environment `public_network_access = "Disabled"`
- Added `private_link_target_id` / `private_link_target_type` to all 4 Front Door backends (lexxi, rd, keycloak, mlflow)
- One Private Link connection (lexxi) was approved on the CAE

**Why it failed:**
- After the changes, ALL Front Door resources (routes, origins, origin groups, WAF policy) stuck at `deploymentStatus: NotStarted`
- Front Door never deployed config to edge PoPs → all requests returned Azure's generic 404 page
- rd/keycloak/mlflow never generated Private Link connection requests (Azure uses shared private endpoint — one per CAE — but that didn't unblock deployment)
- Cycling private link on origins (remove then re-add via REST API) did not fix it

**Current state (after rollback, commit 1bf34ce):**
- `public_network_access = "Enabled"` on CAE (pushed, CI/CD pipeline will apply)
- All 4 Front Door origins have `sharedPrivateLinkResource` cleared (done via REST API before commit)
- Private Link connection on CAE deleted
- Terraform config has no private_link on backends

**Why:** `publicNetworkAccess` cannot be set via standard Azure REST API (field not exposed). Must go through Terraform apply with real credentials (CI/CD pipeline).

**How to apply:** If private link is attempted again, do it fresh on a NEW Front Door profile. Do NOT try to add private link to an existing profile that has routes — the ForceNew deadlock (routes block origin deletion) causes serious issues.
