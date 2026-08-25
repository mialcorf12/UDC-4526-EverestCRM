# EverestCRM Read-Only API Access — Phase 3 Verification Checklist

Change: `everestcrm-read-only-api-access`
Scope: Tasks 3.2–3.10 (negative-path suite, positive-path suite, go-live gate, secret handover)
Manifest: `manifest/package.xml` (Task 3.1 — see below)
Deployment package version: API `67.0`

## Deployment Package Contents (Task 3.1)

`manifest/package.xml` now declares every component created across PR1 and PR2:

| Metadata Type | Member | Origin |
|---|---|---|
| `PermissionSet` | `Everest_CRM_Read_Only` | PR1 |
| `ExternalClientApplication` | `EverestCRM_Integration` | PR2 |
| `ExtlClntAppGlobalOauthSettings` | `EverestCRM_Integration` | PR2 |
| `ExtlClntAppOauthSettings` | `EverestCRM_Integration` | PR2 |
| `ExtlClntAppOauthConfigurablePolicies` | `EverestCRM_Integration` | PR2 |

Verified: root XML element names in the four ECA-family files (`ExternalClientApplication`, `ExtlClntAppGlobalOauthSettings`, `ExtlClntAppOauthSettings`, `ExtlClntAppOauthConfigurablePolicies`) match the `<name>` values above exactly, so the manifest resolves against the correct source-format folders (`externalClientApps`, `extlClntAppGlobalOauthSets`, `extlClntAppOauthSettings`, `extlClntAppOauthPolicies`).

## ⚠️ Deploy Validation Gate Result (blocking finding)

Command run (focused test per tasks forecast, Unit 3):

```bash
sf project deploy validate --manifest manifest/package.xml --target-org Dev --test-level RunLocalTests --wait 30 --json
```

Result: **FAILED — 2 component errors**

```
Error in EverestCRM_Integration - Enter a valid execution user for the OAuth client credentials flow.
Error in Everest_CRM_Read_Only - Unknown user permission: ApiOnlyUser
```

| # | Component | Error | Root cause | Status |
|---|---|---|---|---|
| 1 | `EverestCRM_Integration` (`ExtlClntAppOauthConfigurablePolicies`) | `Enter a valid execution user for the OAuth client credentials flow.` | `clientCredentialsFlowUser=EverestCRM` references a user that does not exist yet in the `Dev` sandbox. Known cross-PR dependency — Tasks 1.5/1.6 (provision + assign `EverestCRM` Integration user) are still unchecked. | **Expected**, not a defect. Resolves once 1.5/1.6 are completed. |
| 2 | `Everest_CRM_Read_Only` (PermissionSet) | `Unknown user permission: ApiOnlyUser` | Task 1.4 (PR1) added `<userPermissions><name>ApiOnlyUser</name></userPermissions>`. `ApiOnlyUser` is **not** a grantable/permission-set user permission in this org's metadata schema (API v67.0, Enterprise Edition) — it is a read-only attribute of the User record tied to license type, not something a PermissionSet can enable. | **Genuine defect** introduced in PR1. Blocks deploy of the entire manifest. |

**Architect's note**: finding #2 is exactly what a manifest-level validation gate exists to catch — a component that individually looked plausible (task 1.4's stated intent was correct: lock the integration user to headless/API-only access) but uses a field that isn't assignable at the permission-set layer. The correct mechanism for "API-only, no UI login" is the **license type** (`Salesforce Integration` license, or `Minimum Access - Salesforce` profile with no full CRM license) selected at Task 1.5 user-provisioning time — not a PermissionSet `userPermissions` entry. Recommended fix for the PR1 follow-up: remove the `<userPermissions>` block from `Everest_CRM_Read_Only.permissionset-meta.xml` entirely; enforce "API-only" purely through the license/profile chosen when provisioning `EverestCRM` (Task 1.5). This is **out of scope for PR3** (PR3 owns manifest + verification docs only) and is called out here so it is not silently absorbed into "deploy failed because of missing user."

**Gate consequence**: Task 3.9 (go-live gate) **cannot pass** until both rows above are resolved:
1. `EverestCRM` user provisioned and assigned the permission set (Tasks 1.5/1.6).
2. The invalid `ApiOnlyUser` user permission is removed from `Everest_CRM_Read_Only` (PR1 follow-up fix — new task recommended below).

Until both land, Tasks 3.2–3.8 below **cannot be executed against a live token** because there is no way to mint an access token bound to `EverestCRM` (no user, and the permission set that would be assigned to it does not deploy). The procedures below are written as an **executable runbook** ready to run the moment the manifest deploys clean; they are not fabricated pass/fail results.

## Task 3.2–3.7 — Negative-Path Test Suite (RED, execute after clean deploy)

Prerequisite: valid access token for `EverestCRM` via:

```bash
curl -s https://<my-domain>.my.salesforce.com/services/oauth2/token \
  -d "grant_type=client_credentials" \
  -d "client_id=<consumer_key>" \
  -d "client_secret=<consumer_secret>" | tee /tmp/token.json
ACCESS_TOKEN=$(python3 -c "import json;print(json.load(open('/tmp/token.json'))['access_token'])")
INSTANCE_URL=$(python3 -c "import json;print(json.load(open('/tmp/token.json'))['instance_url'])")
```

| # | Case | Command | Expected result |
|---|---|---|---|
| 3.2 | Create denied | `curl -s -o /dev/null -w "%{http_code}\n" -X POST "$INSTANCE_URL/services/data/v67.0/sobjects/Account" -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -d '{"Name":"Negative Test"}'` | `403` with body `errorCode: INSUFFICIENT_ACCESS_OR_READONLY`; zero Account rows created |
| 3.3 | Update denied | `curl -s -o /dev/null -w "%{http_code}\n" -X PATCH "$INSTANCE_URL/services/data/v67.0/sobjects/Account/<known_id>" -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -d '{"Name":"Should Not Apply"}'` | `403`; record `Name` unchanged on re-query |
| 3.4 | Delete denied | `curl -s -o /dev/null -w "%{http_code}\n" -X DELETE "$INSTANCE_URL/services/data/v67.0/sobjects/Contact/<known_id>" -H "Authorization: Bearer $ACCESS_TOKEN"` | `403`; record still present on re-query |
| 3.5 | Bulk write denied | `sf data create job --object Account --operation insert --target-org <EverestCRM alias>` + submit one CSV row | Job fails at row/job level with insufficient-access error; `numberRecordsProcessed=0`, `numberRecordsFailed>0` |
| 3.6 | Apex REST denied | `curl -s -o /dev/null -w "%{http_code}\n" "$INSTANCE_URL/services/apexrest/<any @RestResource path>" -H "Authorization: Bearer $ACCESS_TOKEN"` | `403`; no Apex class access exists on the permission set (confirmed statically in PR2 Task 2.6 — no `classAccesses` entries) |
| 3.7 | Flow invocation denied | `curl -s -o /dev/null -w "%{http_code}\n" -X POST "$INSTANCE_URL/services/data/v67.0/actions/custom/flow/<FlowApiName>" -H "Authorization: Bearer $ACCESS_TOKEN" -H "Content-Type: application/json" -d '{"inputs":[{}]}'` | `403`/insufficient-access; no `flowAccesses` entries exist on the permission set (confirmed statically in PR2 Task 2.6) |

Static confirmation already done (does not require a live token): the permission set and all four ECA files contain zero `classAccesses`, `pageAccesses`, `flowAccesses`, or `customPermissions` entries — verified during PR2 (Task 2.6). Rows 3.6/3.7 above are runtime confirmation of that static fact, not a new grant path.

## Task 3.8 — Positive-Path Verification (GREEN)

| Check | Command | Expected result |
|---|---|---|
| Approved object read | `curl -s -o /dev/null -w "%{http_code}\n" "$INSTANCE_URL/services/data/v67.0/query?q=SELECT+Id+FROM+Account+LIMIT+1" -H "Authorization: Bearer $ACCESS_TOKEN"` | `200` with `records` array |
| Out-of-scope object denied | `curl -s -o /dev/null -w "%{http_code}\n" "$INSTANCE_URL/services/data/v67.0/query?q=SELECT+Id+FROM+Case+LIMIT+1" -H "Authorization: Bearer $ACCESS_TOKEN"` | Error response, `errorCode: INVALID_TYPE` (Case is not in the approved object list; no `objectPermissions` entry exists for it) |
| Repeat for all 12 approved standard objects | Same `query` pattern against User, UserRole, Profile, RecordType, Account, Contact, Lead, AccountContactRelation, Opportunity, OpportunityContactRole, Campaign, CampaignMember | `200` for each |
| Custom objects (Practice, Doctor/Staff, uLab ID, Territory) | N/A — Tasks 2.1/2.2 remain BLOCKED; no custom object grants exist in this repo (confirmed absence of `force-app/main/default/objects/` during PR2) | Not testable until custom object API names are confirmed against the target org and PR2 is amended |

## Task 3.9 — Go-Live Security Gate

Per spec requirement "Auditable Verification": *"Security sign-off MUST NOT be granted without a passing run"* covering create, update, delete, bulk write, Apex, and Flow denial.

**Current gate status: 🔴 BLOCKED — sign-off MUST NOT be granted.**

Blocking items, in order:
1. Fix `Everest_CRM_Read_Only` PermissionSet: remove the invalid `<userPermissions><name>ApiOnlyUser</name></userPermissions>` block (deploy-blocking defect found above).
2. Complete Task 1.5: provision `EverestCRM` user on an Integration-class license (or `Minimum Access - Salesforce` profile), zero admin permissions.
3. Complete Task 1.6: assign `Everest_CRM_Read_Only` to `EverestCRM`; confirm zero `Modify All Data` / `View All Data`.
4. Re-run `sf project deploy validate --manifest manifest/package.xml --test-level RunLocalTests` → must return `0` component errors.
5. Execute Tasks 3.2–3.8 against a live token and record actual HTTP status codes/error codes for every row.
6. Resolve Tasks 2.1/2.2 (custom object API name confirmation) or explicitly accept them as descoped follow-up before sign-off — do not sign off silently omitting the custom-object requirement from the original spec.
7. Only after all rows in 3.2–3.8 show the expected denial/success codes, and this checklist is attached as the audit artifact, may security sign-off be granted.

## Task 3.10 — Secret Handover via Key Vault

Per spec requirement "Secret Custody": the client secret MUST be delivered only through the shared key vault, and MUST NOT appear in repository files, metadata, logs, email, or chat.

Handover procedure:
1. Retrieve the ECA consumer key/secret from Setup → External Client Apps → `EverestCRM Integration` → OAuth settings, in a private admin session (not shared screen, not shell history).
2. Store the following fields as a single vault entry (vault system: **TBD — Open Question #3 from the spec is still unresolved; do not proceed with handover until the vault system is named**):
   - My Domain URL
   - Org ID
   - Environment type (sandbox/production)
   - Consumer Key (Client ID)
   - Consumer Secret (Client Secret)
   - Daily API call limit / rate-limit guidance for the polling frequency EverestCRM will use
3. Share the vault entry link (not the values) with the Digital Marketing Manager per the Jira user story's Scenario 4.
4. Confirm no plaintext copy exists in: this repository, chat threads, email, or logs (`sf` CLI output redacts access tokens by default — verified in this session's `sf org list --json` output, which shows `"accessToken": "[REDACTED]..."`).
5. On suspected exposure: rotate the consumer secret in Setup, revoke all issued tokens, update the vault entry, and note the rotation date/reason in the vault's audit trail.

**Open blocker carried from spec**: the key vault system (1Password, Azure Key Vault, or other) is not yet named. This checklist cannot be marked complete for Task 3.10 until that decision is made — tracked as an open question, not fabricated as resolved.

## Summary of Phase 3 Findings

| Finding | Severity | Owner | Status |
|---|---|---|---|
| Manifest correctly enumerates all PR1+PR2 components | — | PR3 | ✅ Done |
| `ApiOnlyUser` is not a valid PermissionSet user permission — blocks deploy | High | PR1 follow-up | 🔴 Open |
| `EverestCRM` user not yet provisioned — blocks ECA deploy and all live testing | High | PR1 (Tasks 1.5/1.6) | 🔴 Open |
| Custom object grants (Practice, Doctor/Staff, uLab ID, Territory) not present | Medium | PR2 (Tasks 2.1/2.2) | 🔴 Open (pre-existing, carried forward) |
| Key vault system unnamed — secret handover cannot be finalized | Medium | Spec owner | 🔴 Open (pre-existing, carried forward) |
| Negative/positive test procedures documented and ready to execute | — | PR3 | ✅ Done (runbook only, not yet executed live) |
