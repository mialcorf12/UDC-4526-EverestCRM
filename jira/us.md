# User Story: Configure Read-Only Salesforce API Access for EverestCRM Integration

## Description
As a **Digital Marketing Manager**, I want a **dedicated read-only Salesforce integration setup (Integration User, Permission Set, and External Client App using OAuth 2.0 Client Credentials Flow) for EverestCRM**, so that **we can switch EverestCRM from static data to live, connected data for reporting, analysis, and visualization without risking unauthorized data modifications**.

## Background
EverestCRM currently relies on static data exports for reporting and marketing analytics. To enable real-time dashboarding and visualization while adhering to enterprise security and principle of least privilege, we need a secure, read-only API connection between Salesforce and EverestCRM using modern OAuth 2.0 Client Credentials Flow.

## Acceptance Criteria

### Scenario 1 — Successful OAuth 2.0 Client Credentials Authentication (Happy Path)
```gherkin
Given an External Client App (or Connected App) configured with OAuth 2.0 Client Credentials Flow
  And assigned to run as the dedicated "EverestCRM" Integration User
When EverestCRM requests an access token from the Salesforce OAuth token endpoint (/services/oauth2/token) using valid client credentials
Then Salesforce returns a 200 OK HTTP response with a valid OAuth access token
  And the token context assumes the permissions assigned to the "EverestCRM" user.
```

### Scenario 2 — Read-Only Data Querying via REST & Bulk API 2.0
```gherkin
Given EverestCRM presents a valid access token for the "EverestCRM" Integration User
When EverestCRM executes SOQL queries or Bulk API 2.0 jobs against allowed standard and custom objects:
  | Object Category                | Included Entities / Objects                                       |
  | Core Identity & Security       | User, UserRole, Profile, RecordType                               |
  | CRM Core                       | Account, Contact, Lead, AccountContactRelation                    |
  | Sales & Marketing              | Opportunity, OpportunityContactRole, Campaign, CampaignMember     |
  | Custom Entities (Read-Only)    | Practice, Doctor/Staff Relationships, uLab/Customer IDs, Territory|
Then Salesforce returns the requested records and fields
  And system audit fields (CreatedDate, LastModifiedDate, SystemModstamp, Salesforce IDs) are accessible.
```

### Scenario 3 — Strict Read-Only Policy Enforcement (Negative Path)
```gherkin
Given EverestCRM presents a valid access token for the "EverestCRM" Integration User
When EverestCRM attempts a Create (INSERT), Update (UPDATE), Upsert (UPSERT), or Delete (DELETE) DML operation via API
Then Salesforce rejects the API request with HTTP 403 Forbidden or "INSUFFICIENT_ACCESS_OR_READONLY" error
  And no record modification occurs in the database.
```

### Scenario 4 — Secure Credential Delivery & Handover
```gherkin
Given the External Client App and Integration User configuration is complete and verified
When the Salesforce Admin performs the technical handover to the Digital Marketing Manager
Then credentials (My Domain URL, Org ID, Environment Type, Client ID, Client Secret, API limits) are shared securely via an encrypted vault (e.g., 1Password / Azure Key Vault)
  And no secrets or credentials are sent in plain text (chat or unencrypted email).
```

## Salesforce Non-Functional AC
- [ ] **Principle of Least Privilege**: Permission set `Everest CRM Read Only` grants strictly Read/View All permissions on target objects; zero Create/Edit/Delete or Modify All Data permissions.
- [ ] **Bulk & Query Performance**: Integration queries rely on indexed fields (`Id`, `CreatedDate`, `SystemModstamp`, External IDs) to avoid full table scans during sync windows.
- [ ] **Security & OAuth Flow**: Uses External Client App (ECA) or Connected App with OAuth 2.0 Client Credentials Flow; IP restrictions/whitelisting configured if required by org policy.
- [ ] **FLS & Field Security**: Read access explicitly granted to business-critical standard fields (Identity, Contact, Address, Status, Conversion, Opportunity) and specified custom fields.
- [ ] **Auditability**: All API activity is audited under the dedicated `EverestCRM` Integration User.

## Out of Scope
- Bidirectional data synchronization or write-back capability into Salesforce.
- Real-time event streaming (Platform Events, Change Data Capture, or Outbound Messages) — pull/query model only.
- User creation or SSO configuration for EverestCRM end-users inside Salesforce.
- Custom Apex REST endpoints or Apex trigger development.

## Story Points
**5** — Moderate Salesforce administration complexity involving Integration User creation, comprehensive Permission Set FLS/CRUD mapping across ~10 objects, External Client App / OAuth 2.0 Client Credentials Flow configuration, and security testing.

## Subtasks
- [ ] **1. Integration User Setup**: Create dedicated Integration User `EverestCRM` with `Minimum Access - Salesforce` profile or Salesforce Integration license.
- [ ] **2. Permission Set Authoring**: Create Permission Set `Everest CRM Read Only` and assign Read/View All access to required Standard & Custom objects and fields.
- [ ] **3. Permission Set Assignment**: Assign `Everest CRM Read Only` permission set to `EverestCRM` integration user.
- [ ] **4. OAuth Application Setup**: Configure External Client App (or Connected App) with OAuth 2.0 Client Credentials Flow, setting `EverestCRM` as the Run-As user and granting API scopes (`api`, `refresh_token`, `offline_access`).
- [ ] **5. Integration Testing (Positive)**: Authenticate via Postman / sf CLI using Client Credentials and execute sample SOQL & Bulk API 2.0 read queries against Accounts, Contacts, Leads, Opportunities, and Campaigns.
- [ ] **6. Integration Testing (Negative)**: Test DML mutation requests (POST/PATCH/DELETE) to confirm strict 403 Read-Only enforcement.
- [ ] **7. Security Handover**: Securely deposit My Domain URL, Org ID, Client ID, Client Secret, environment details, and daily API quota guidelines into secure vault for handover.

## Open Questions
- [ ] Confirm specific API names for custom objects/fields (e.g., Practices, uLab Customer ID, Doctor/Staff relationships).
- [ ] Confirm whether IP Whitelisting or Trusted IP ranges are mandatory for the External Client App security policy in this org.
