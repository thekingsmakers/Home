# Reusable Multi-Tenant SaaS Architecture Blueprint

## Purpose

This document extracts the architectural methods implemented by TKM X Control into a reusable blueprint for building another business-to-business SaaS platform with the same operating model.

It describes patterns rather than prescribing the TKM X Control product domain. Replace devices, policies, packages, and agents with the resources managed by the new system.

The target model supports:

- a public website and sign-in experience;
- a tenant administration portal;
- a separate platform-operator portal;
- shared infrastructure with strict tenant isolation;
- subscription plans, entitlements, trials, licenses, usage, and billing;
- remote agents or workers that connect outbound to the platform;
- durable asynchronous operations;
- audited support access and tenant lifecycle management;
- PostgreSQL production storage and a lightweight local/test database;
- horizontal application scaling and independent worker scaling.

## 1. Architectural principles

1. **The authenticated identity selects the tenant.** Never accept a tenant identifier in a header or request body as authority to change scope.
2. **Platform administration and tenant administration are different security domains.** Give them separate routes, roles, navigation, and authorization rules.
3. **Every tenant-owned row carries `TenantId`.** Enforce it in queries and, in PostgreSQL, with row-level security.
4. **Commands are durable records, not transient HTTP calls.** The API records intent; workers later claim and execute it.
5. **Mutations are auditable and idempotent.** Important writes include actor, timestamp, justification, correlation ID, and an idempotency key where retries are possible.
6. **Licensing is enforced at the API boundary and again in critical domain services.** Hiding a menu item is not enforcement.
7. **Credentials are displayed once, stored only as hashes or encrypted values, and rotated independently.**
8. **Administrative support access is explicit, time-bound, consent-aware, and dual-actor audited.**
9. **The UI reads real APIs and represents loading, empty, error, pending, blocked, and terminal states.**
10. **Production operations fail closed.** Unsafe secrets, placeholder passwords, unknown billing states, invalid signatures, and ambiguous worker results must not silently succeed.

## 2. System context

```text
 Public users        Tenant administrators        Platform operators
      |                       |                           |
      +-----------------------+---------------------------+
                              |
                    Web application / gateway
                   React or another SPA framework
                              |
                 Same-origin HTTPS REST/WebSocket
                              |
                 Modular application API service
       +----------------------+-----------------------+
       |                      |                       |
 Identity/tenant        Domain modules         Platform control
 authorization          and workflows          plane/licensing
       |                      |                       |
       +----------------------+-----------------------+
                              |
                   Durable relational database
                   PostgreSQL in production
                              |
             +----------------+----------------+
             |                                 |
       Outbound endpoint agents          Outbound worker pool
       or customer connectors            for shared automation
             |                                 |
       Managed resources                   External systems
```

No inbound connection to an endpoint agent is required. The agent initiates TLS connections, sends inventory and heartbeats, receives work signals, claims durable work, and reports structured results.

## 3. Recommended repository and application structure

```text
src/
  Server/
    BuildingBlocks/
      Auth/             authentication, tenant context, permissions
      Audit/            immutable administrative event recording
      Database/         connection factory, migrations, production schema
      Jobs/             durable queue, leases, idempotency, result handling
      Security/         encryption, hashing, secret protection
    Modules/
      Identity/
      Tenants/
      Licensing/
      Billing/
      Agents/
      Resources/        replace with the new product's core domain
      Operations/
      Reporting/
      Platform/
  Web/
    src/api/            typed API client
    src/components/     layouts, guards, shared controls
    src/pages/public/
    src/pages/tenant/
    src/pages/platform/
  Agent/                endpoint or customer-site service
  AgentShared/          versioned work/result contracts
  Worker/               optional Linux/container automation pool
tests/
  Unit/
  Integration/
  Security/
  EndToEnd/
```

The initial backend can remain a modular monolith. This keeps transactions and deployment straightforward while enforcing module boundaries in code. Extract a module into a service only when its independent scale, availability, or ownership requirements justify the operational cost.

## 4. Application/API architecture

### 4.1 API route boundaries

Use distinct route families:

| Boundary | Example prefix | Intended caller |
|---|---|---|
| Public | `/api/public/v1/*` | Anonymous browser or public integration |
| Tenant | `/api/v1/*` | Authenticated tenant user or approved support session |
| Platform | `/api/platform/v1/*` | Platform operator only |
| Agent | `/agent/v1/*` | Enrolled machine identity |
| Worker | `/worker/v1/*` | Enrolled execution worker |
| Health | `/health`, `/ready` | Load balancer and operations |

Do not rely only on controller annotations. Add centralized boundary middleware so every route is evaluated consistently.

### 4.2 Request pipeline order

The order matters:

1. global exception handling and correlation ID;
2. trusted forwarded-header processing;
3. restrictive CORS;
4. authentication for bearer, agent, or worker credentials;
5. authorization-denial audit capture;
6. platform-versus-tenant portal boundary;
7. subscription and module-entitlement enforcement;
8. role/permission enforcement for mutations;
9. controller/domain execution;
10. Swagger/OpenAPI and health endpoints as deliberately allowed exceptions.

Return consistent errors:

```json
{
  "error": "Human-readable explanation",
  "code": "STABLE_MACHINE_CODE",
  "correlationId": "request-correlation-id"
}
```

Use `401` for missing/invalid authentication, `403` for authenticated but unauthorized callers, `402` or `403` for entitlement denial according to the public contract, `404` to conceal cross-tenant resources, `409` for lifecycle/concurrency conflicts, and `422` or `400` for validation failures.

### 4.3 Authentication tokens

A tenant JWT should minimally contain:

```text
sub          immutable user ID
email        display/audit identity
tenant_id    immutable tenant ID
role         effective role
jti          token identifier
iat/nbf/exp  issue, validity, and expiry timestamps
```

A direct platform token binds to a reserved service tenant, for example `tenant-platform`, and contains a platform role. It is rejected from ordinary tenant operational routes.

A support-session token contains two identities:

```text
sub                    effective support subject
tenant_id              target customer tenant
role                    restricted effective tenant role
platform_actor_id       real platform operator
platform_actor_email    real platform operator email
support_session_id      audited session record
support_mode            ReadOnly or ReadWrite
exp                     short session expiry
```

Audit records must preserve both the effective tenant actor and the real platform actor.

### 4.4 Authorization model

Use permissions for actions and roles as permission bundles.

Examples:

```text
Tenant Administrator
  users.manage, roles.manage, resources.manage, automation.manage,
  billing.read, settings.manage, audit.read

Operator
  resources.read, resources.execute, jobs.read

Approver
  approvals.read, approvals.decide, audit.read

Read Only
  resources.read, reports.read

Platform Owner
  platform.*
```

For scoped users, add explicit resource/group scope tables. Authorization becomes:

```text
authenticated
AND token tenant matches resource tenant
AND role grants permission
AND resource is inside the user's assigned scope
AND subscription grants the module
AND tenant/lifecycle state permits the operation
```

### 4.5 Frontend architecture

Maintain three layouts:

- public marketing/onboarding layout;
- tenant console layout;
- platform operations layout.

The client should have one typed API layer responsible for:

- attaching the correct access token;
- same-origin base paths;
- serializing query parameters safely;
- parsing the standard error envelope;
- converting `401` into session expiration;
- never converting `403`, `402`, or `409` into generic network failures;
- supporting cancellation and request timeouts;
- exposing pagination types consistently.

Every data view should implement these states explicitly:

```text
initial loading -> loaded data | empty data | recoverable error
mutation idle -> submitting -> success | field error | conflict | failure
async operation -> pending approval -> queued -> dispatched -> running
                -> verified success | failed | expired | cancelled | unknown
```

## 5. Multi-tenant database design

### 5.1 Core tenancy tables

```text
Tenants
  Id, Name, Slug, Status, PlanId, DeviceLimit/ResourceLimit,
  EntitlementsJson, TrialEndsAt, SubscriptionExpiresAt,
  OnboardingState, SuspendedAt, SoftDeletedAt, OffboardedAt,
  CreatedAt, UpdatedAt

TenantSettings
  TenantId, authentication policy, approval policy,
  heartbeat/polling configuration, support-consent flags,
  retention configuration

Users
  Id, TenantId, Email, PasswordHash/external subject,
  RoleId, Status, MFA state, invitation state, timestamps

TenantRoles
TenantRolePermissions
UserGroupScopes
UserResourceScopes
```

Use immutable surrogate IDs for references. Slugs and emails are mutable business identifiers and should have normalized unique indexes.

### 5.2 Tenant-owned domain tables

Every tenant-owned table includes `TenantId`, including join tables and operation-result tables. A typical family is:

```text
Resources               current managed objects
ResourceInventory       collected child inventory
Groups / GroupMembers   targeting and scope
Artifacts               metadata for deployable content
ContentManifests        immutable hash and storage identity
Campaigns               requested bulk operation
CampaignTargets         frozen target snapshot
CampaignResults         per-target result and evidence
Policies / PolicyVersions / PolicyTargets / PolicyResults
Scripts / ScriptVersions / ScriptApprovals / ExecutionRequests
DurableJobs             authoritative asynchronous work queue
AuditLogs               append-only administrative trail
OutboxMessages          transactionally emitted integration events
```

### 5.3 Platform control-plane tables

```text
PlatformPlans           active commercial catalog
PlatformPlanVersions    immutable commercial history
LicenseKeys             issued/revoked key records and hashes
TenantSubscriptions     provider state and billing period
BillingLedger           charges, credits, status, reconciliation metadata
BillingWebhookEvents    deduplication and ordering state
LicenseUsage            current usage aggregate
LicenseUsageEvents      append-only usage changes
SupportSessions         consent, actor, tenant, mode, expiry, termination
TenantAdminInvitations  one-time onboarding credentials
PlatformReports         asynchronous report state
PlatformReleases        signed agent release catalog
PlatformSettings        global operational controls
WorkerHeartbeats        current worker health
```

Platform tables that refer to customer data should still carry `TenantId` where meaningful, both for isolation and forensic querying.

### 5.4 PostgreSQL row-level security

Application queries must always include `TenantId`, but production PostgreSQL should add a second boundary:

```sql
ALTER TABLE Resources ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_policy ON Resources
USING (TenantId = current_setting('app.current_tenant_id', true))
WITH CHECK (TenantId = current_setting('app.current_tenant_id', true));
```

At the beginning of a tenant transaction:

```sql
SET LOCAL app.current_tenant_id = 'validated-tenant-id';
```

Important rules:

- set tenant context from validated identity claims, never request input;
- apply RLS to child, join, history, audit, job, and usage tables too;
- use a distinct, tightly controlled platform connection path for legitimate cross-tenant operations;
- do not run the application as a PostgreSQL role that bypasses RLS;
- test cross-tenant reads, inserts, updates, deletes, joins, and aggregate queries;
- use `WITH CHECK` to prevent inserting a row into another tenant;
- decide deliberately whether table owners are subject to `FORCE ROW LEVEL SECURITY`.

### 5.5 Transactions and concurrency

Use one transaction for workflows such as tenant provisioning:

```text
create tenant
create default settings
snapshot plan entitlements
create initial subscription/trial
create administrator invitation
create initial role assignments
write audit event
commit
```

Use optimistic concurrency for editable definitions:

```sql
UPDATE Policies
SET CurrentVersion = CurrentVersion + 1
WHERE Id = @Id AND TenantId = @Tenant AND CurrentVersion = @Expected;
```

If the affected-row count is zero, return `409 Conflict`.

Use unique constraints for idempotency and natural invariants, not pre-checks alone. Translate constraint violations into stable `409` responses.

### 5.6 Audit and outbox

Audit records should include:

```text
Id, TenantId, Timestamp, Action, EntityType, EntityId,
ActorUserId, ActorEmail, ActorRole,
PlatformActorId, PlatformActorEmail, SupportSessionId,
IPAddress, CorrelationId, Details
```

Write the business mutation and audit record in the same transaction for high-risk operations.

For external events, write an `OutboxMessages` row in that same transaction. A background processor publishes it and marks it complete. This prevents a committed database change from losing its corresponding message.

## 6. Durable asynchronous operation model

HTTP requests should enqueue work and return an operation identifier. The queue is a database table so work survives application restarts.

### 6.1 Work envelope

```json
{
  "schemaVersion": "2.0",
  "operationId": "uuid",
  "tenantId": "tenant-id",
  "resourceId": "target-id",
  "executionRoute": "Agent",
  "operationType": "ApplyPolicy",
  "payloadJson": "{...}",
  "executionContext": "System",
  "timeoutSeconds": 300,
  "expiresAt": "ISO-8601",
  "notBefore": "ISO-8601 or null",
  "correlationId": "business-operation-id"
}
```

### 6.2 Job state machine

```text
Queued
  -> Dispatched
  -> Running
  -> Succeeded | Failed | Expired | Cancelled | OutcomeUnknown
```

`OutcomeUnknown` is important: if a non-idempotent mutation was dispatched and the worker disappeared, do not automatically report failure or replay it. Reconcile actual target state first.

Recommended columns:

```text
Id, TenantId, TargetId, JobType, PayloadJson, Status,
ExecutionRoute, IdempotencyKey, CorrelationId,
CreatedAt, NotBefore, DispatchedAt, StartedAt, CompletedAt, ExpiresAt,
LeaseOwner, LeaseExpiresAt, RetryCount, MaxRetries,
ExitCode, Stdout, ErrorMessage,
BeforeEvidence, AfterEvidence, VerificationOutcome,
ChangeOccurred, RebootRequired, DurationMs, Stage
```

### 6.3 Queue invariants

- Claim with an atomic conditional update or `SELECT ... FOR UPDATE SKIP LOCKED`.
- Lease work; do not permanently assign it merely by reading it.
- Renew leases for long operations.
- Cap retries and use exponential backoff.
- Deduplicate active jobs with `(TenantId, TargetId, IdempotencyKey)`.
- Freeze targets when an approval request is created.
- Freeze the reviewed payload/hash so approval cannot be reused after content changes.
- Reject late, duplicate, wrong-route, or conflicting terminal results.
- Limit and redact captured output.
- Store before/after evidence and verification separately from process exit code.

## 7. Agent architecture

### 7.1 Enrollment

1. Tenant administrator creates a short-lived, single-use enrollment token.
2. The installer receives server URL, role, token, and optional release channel.
3. Agent sends token plus machine identity facts to `/agent/v1/enroll`.
4. Server validates token scope, expiry, use count, tenant state, and licensed capacity.
5. Server creates an immutable agent identity and returns a generated credential.
6. Server stores only the credential hash.
7. Agent stores the identity using the operating system's protected credential store.
8. The enrollment token is consumed and an audit event is committed.

An agent identity is separate from a human identity:

```text
AgentId, TenantId, Role, SubjectId, CredentialHash,
CredentialExpiresAt, Status, Version, CreatedAt, RevokedAt
```

### 7.2 Runtime communication

The agent uses outbound communication only:

```text
startup
  -> load protected identity or enroll
  -> configure Agent credential header
  -> upload inventory
  -> send heartbeat
  -> open WebSocket work-signal channel
  -> long-poll durable queue as recovery path
  -> claim one job
  -> execute and verify
  -> upload structured result
  -> buffer result locally if disconnected
  -> retry result delivery without re-executing work
```

The WebSocket is only a wake-up signal. The durable database queue remains authoritative. Losing the socket must not lose work.

### 7.3 Credential rotation and revocation

- Rotate before expiry using the current valid credential.
- Return the new secret once and atomically replace the server hash.
- Persist the new local identity safely before discarding the old credential where possible.
- Support immediate revocation and unenrollment.
- A retired resource cannot claim new work.
- Cancel queued work during retirement and mark already-dispatched mutations uncertain until reconciled.

### 7.4 Execution safety

For each operation:

1. validate schema and size limits;
2. validate immutable content hash/signature;
3. capture pre-state;
4. execute with an explicit timeout and identity;
5. bound stdout/stderr and redact secrets;
6. capture post-state;
7. calculate verification independently of exit code;
8. report structured evidence;
9. refresh inventory after verified changes.

Run machine-wide operations under a service identity. For user-context work, use an explicit session-aware broker rather than assuming the service can impersonate the interactive user safely.

### 7.5 Release management

- Publish signed immutable artifacts.
- Record version, channel, SHA-256, signature status, creation date, and rollout status.
- Pin the expected hash in the installer/update contract.
- Download to a temporary location and verify before replacement.
- Preserve configuration and identity during upgrades.
- Stop, replace, start, health-check, and roll back atomically.
- Maintain Stable, Beta, and Preview channels with tenant-level selection.

## 8. Licensing, subscription, and billing architecture

### 8.1 Separate the concepts

Do not collapse everything into one `Plan` column:

- **Plan catalog:** commercial definition and included modules.
- **Plan version:** immutable history of price/quota/entitlement changes.
- **Subscription:** provider status, billing period, external IDs.
- **Entitlement snapshot:** what the tenant currently has access to.
- **License key:** optional offline/manual activation credential.
- **Usage:** current aggregate and append-only usage events.
- **Ledger:** financial records and reconciliation status.

### 8.2 Plan model

```text
PlatformPlans
  Id, Name, Description, ResourceLimit, BillingCycle,
  IncludedModulesJson, IsTrial, IsActive, CurrentVersion

PlatformPlanVersions
  PlanId, Version, immutable commercial fields,
  EffectiveAt, CreatedBy, ChangeReason
```

When a tenant subscribes or converts, copy the effective included modules and limits into a tenant entitlement snapshot. Do not let later catalog edits silently change existing contractual access unless that is an explicit migration.

### 8.3 Subscription state model

```text
Trialing -> Active -> PastDue -> Suspended -> Cancelled
    |         |          |           |
    +---------+----------+-----------+-> Expired
```

Define behavior for every state. An example policy:

| State | Reads | Mutations | New enrollment | Exports/audit |
|---|---:|---:|---:|---:|
| Trialing/Active | Yes | Yes | Within quota | Yes |
| PastDue within grace | Yes | Yes or limited | Optional | Yes |
| Suspended after grace | Inventory only | No | No | Yes |
| Cancelled/Expired | Retained read-only | No | No | Yes during retention |

Unknown provider states must fail closed and alert operations.

### 8.4 License keys

Generate a high-entropy random secret. Store:

```text
Id, TenantId, PlanId, KeyHash, KeyHint,
ResourceLimit, IssuedAt, ExpiresAt, Status,
IssuedBy, ActivatedAt, RevokedAt, RevokedBy
```

Return the full key only once. Validate by hashing the supplied key and performing constant-time comparison. Validation also checks tenant status, plan status, expiry, revocation, quota, and subscription rules.

Never store or log the plaintext key.

### 8.5 Webhook processing

Billing webhooks require:

- provider signature verification over the raw request body;
- event-ID uniqueness;
- payload hash storage;
- event occurrence timestamp, not only receive timestamp;
- stale/out-of-order event rejection;
- one transaction for event record, subscription update, entitlement update, and audit;
- an explicit processing outcome such as `Applied`, `Duplicate`, `StaleIgnored`, or `Rejected`.

Do not trust a webhook's plan ID without matching it to an active internal catalog plan.

### 8.6 Usage and reconciliation

Record usage changes as events and periodically compute aggregates:

```text
LicenseUsageEvents: TenantId, Metric, Delta, SourceId, Timestamp
LicenseUsage:       TenantId, Metric, CurrentValue, UpdatedAt
BillingLedger:      TenantId, Period, Amount, Currency, Status, Notes
```

Reconciliation compares subscription limits, actual usage, ledger entries, and provider state. It must be idempotent for a billing period.

## 9. Tenant lifecycle and management

### 9.1 Tenant state machine

```text
Provisioning -> Active <-> Suspended -> SoftDeleted -> Offboarded
      |           |
      +-----------+-> TrialExpired / SubscriptionExpired
```

Recommended meanings:

- `Provisioning`: atomic setup is not complete; login is denied.
- `Active`: normal access subject to subscription entitlements.
- `Suspended`: login/mutations restricted, data retained.
- `SoftDeleted`: hidden from ordinary views, recoverable during retention.
- `Offboarded`: credentials revoked, work cancelled, exports completed, deletion/retention policy initiated.

Transitions are commands with validation, justification, actor, timestamp, audit event, and a compare-and-set state update. Do not allow arbitrary status assignment.

### 9.2 Atomic provisioning

Provisioning input:

```text
tenant name, unique slug, plan, quota,
administrator email, locale/time zone, optional trial choice
```

Transaction:

1. normalize and validate name/slug/email;
2. verify catalog plan and quota rules;
3. insert tenant in `Provisioning` state;
4. create settings and entitlement snapshot;
5. create subscription or trial;
6. create roles and administrator membership;
7. issue a hashed, expiring, one-use invitation;
8. write platform audit entry;
9. set onboarding state and tenant state to active;
10. commit and return the invitation secret once.

Invitation acceptance sets the password or external identity, enrolls MFA when required, consumes the token, and activates the membership.

### 9.3 Suspension/reactivation

Suspension should:

- require reason and appropriate platform permission;
- reject the reserved platform tenant;
- use allowed-state transitions only;
- prevent new tenant sessions and mutations;
- preserve data and optionally keep agent inventory heartbeats;
- audit the platform actor and affected tenant.

Reactivation verifies billing/license eligibility before restoring access.

### 9.4 Soft deletion and offboarding

Soft deletion records deletion time and reason without immediately erasing data.

Offboarding is an orchestrated workflow:

1. freeze tenant mutations;
2. generate or confirm final export;
3. revoke user sessions, invitations, agent identities, workers, and API credentials;
4. cancel queued work;
5. mark dispatched mutations `OutcomeUnknown` where necessary;
6. disable subscriptions and licenses;
7. detach or quarantine content;
8. record retention/deletion deadlines;
9. write immutable platform and tenant audit events;
10. move tenant to `Offboarded`.

Hard deletion should be a separate retention job, not an API button.

### 9.5 Platform tenant-management interface

Provide:

- overview metrics and health;
- searchable tenant directory;
- tenant overview, memberships, subscription, license, usage, connectors, and audit tabs;
- controlled suspend/reactivate/delete/offboard actions;
- administrator invitation management;
- time-bound support sessions;
- exports and reports;
- worker/connector health;
- full actor and reason visibility.

## 10. Controlled support access

Support access flow:

```text
operator selects tenant
  -> supplies ticket/justification and duration
  -> platform checks tenant consent policy
  -> creates SupportSession
  -> issues short-lived dual-actor token
  -> tenant UI displays support banner/countdown
  -> every action logs both identities
  -> operator or tenant can terminate immediately
  -> token becomes invalid when session is expired/terminated
```

Read-only support tokens must be blocked from mutation routes in centralized middleware, not merely by disabling buttons.

Recommended duration is 15–60 minutes, with no refresh token.

## 11. Content and distribution model

For systems that distribute artifacts:

- store content outside replaceable application build directories;
- address content by SHA-256;
- separate artifact metadata from immutable content manifests;
- scan/validate file type and signatures before publication;
- authorize each download by tenant, target, content hash, expiry, byte limit, and attempt limit;
- support central storage and optional customer-site distribution points;
- make fallback to internet/central sources an explicit reviewed policy;
- record transfer attempts and verification results.

In a multi-node deployment, content storage must be shared or object-backed. Separate local disks will produce inconsistent downloads.

## 12. Deployment topology and scaling

### 12.1 Application tier

Run multiple identical API instances behind a reverse proxy or load balancer:

- same application release;
- same PostgreSQL database;
- same JWT signing configuration;
- shared content/object storage;
- trusted forwarded-header configuration;
- `/health` for process liveness and `/ready` for dependency readiness;
- no sticky sessions when all identity is in bearer tokens and durable storage.

### 12.2 Worker tier

Workers scale separately from API instances. Each worker:

- enrolls with a one-time token;
- receives a unique credential;
- polls outbound;
- claims jobs through database leases;
- reports heartbeat, version, capacity, and readiness;
- has network reachability to the resources it may manage.

Two or more workers competing through leases provide availability without an inbound worker load balancer.

### 12.3 Secrets and configuration

Required production secrets commonly include:

```text
database credential
JWT signing key or asymmetric signing certificate
application field-encryption key
billing webhook secret
OIDC client secrets
artifact-signing trust configuration
object-storage credentials
```

Reject startup when production contains placeholders, short secrets, default database passwords, disabled signature enforcement, or an unsupported billing provider.

## 13. Observability and operations

Expose:

- correlation IDs in errors and logs;
- structured request, actor, tenant, and operation fields;
- database latency;
- durable queue depth by state and age;
- oldest queued job age;
- agent/worker heartbeat freshness;
- connector readiness;
- report/outbox failures;
- process memory and runtime version;
- licensing and billing reconciliation alerts.

Never log bearer tokens, enrollment tokens, license keys, passwords, vault values, webhook secrets, or raw authorization headers. Apply output redaction to remote command results.

## 14. Testing strategy

### Layer 1: unit tests

- state transitions;
- entitlement mapping;
- permission evaluation;
- hash/signature behavior;
- command builders;
- result interpretation;
- retry and expiry calculations.

### Layer 2: database integration tests

- atomic provisioning rollback;
- unique constraints and concurrency;
- idempotent webhook/outbox handling;
- durable job leasing;
- late/duplicate results;
- PostgreSQL query compatibility;
- RLS read/write isolation.

SQLite is useful for fast tests, but it must not be the only database test because its grouping, typing, migration, and concurrency behavior differs from PostgreSQL.

### Layer 3: API contract tests

- `401` versus `403` behavior;
- tenant/platform route boundaries;
- validation and stable errors;
- pagination and filtering;
- OpenAPI generation;
- content upload/download;
- license enforcement.

### Layer 4: stateful end-to-end tests

- provision tenant and accept invitation;
- authenticate tenant administrator;
- enroll and rotate an agent;
- create, approve, execute, and verify work;
- issue/validate/revoke a license;
- trial extension and conversion;
- support session creation and termination;
- suspend/reactivate/offboard tenant;
- billing webhook and reconciliation;
- asynchronous report completion.

Each run should provision isolated test identities instead of assuming permanent seed data. Use unique emails/slugs and clean up or expire fixtures through supported lifecycle APIs.

### Layer 5: adversarial tests

- forged, expired, wrong-audience, and `alg:none` tokens;
- cross-tenant IDOR attempts;
- SQL/XSS payloads;
- duplicate and out-of-order webhooks;
- repeated submissions/idempotency races;
- worker crash after dispatch;
- tampered artifacts and results;
- stale leases;
- subscription expiration during queued work;
- logging/output secret leakage.

## 15. Suggested implementation sequence

### Phase 1: foundations

1. Define tenant, user, role, permission, and audit models.
2. Implement authentication and tenant context from validated claims.
3. Establish tenant/platform/agent route boundaries.
4. Create PostgreSQL schema and RLS tests.
5. Implement standard errors, correlation IDs, health, and OpenAPI.

### Phase 2: tenant lifecycle

1. Atomic tenant provisioning.
2. One-time administrator invitations.
3. Tenant settings and MFA policy.
4. Suspend/reactivate/soft-delete/offboard state machine.
5. Platform tenant directory and detail views.

### Phase 3: commercial controls

1. Versioned plan catalog.
2. Entitlement snapshots and middleware.
3. Trial lifecycle.
4. License issue/validate/revoke.
5. Subscription webhook ingestion.
6. Usage and billing ledger reconciliation.

### Phase 4: durable work

1. Durable jobs and outbox.
2. Target freezing and approvals.
3. Agent enrollment and credentials.
4. Heartbeats, WebSocket signals, and long-poll recovery.
5. Structured results, verification, and uncertain outcomes.

### Phase 5: product modules

Build domain features on the established tenancy, entitlement, job, audit, and agent primitives. Avoid inventing a separate queue or authorization model for every module.

### Phase 6: operations and hardening

1. Support impersonation.
2. Reports and exports.
3. Release channels and signed upgrades.
4. Worker pools and connectors.
5. Full PostgreSQL, API, E2E, and adversarial suites.
6. Multi-node deployment and restore drills.

## 16. Decisions to make for the new product

Before implementation, explicitly decide:

1. Shared database with RLS, schema-per-tenant, or database-per-tenant?
2. JWT symmetric signing or asymmetric/OIDC identity?
3. Which actions require two-person approval?
4. Which subscription states retain read access?
5. What is the source of truth for usage and billing?
6. Are agents required, optional, or replaced by API integrations?
7. Can operations be safely retried, or do they require reconciliation?
8. What evidence proves an operation succeeded?
9. What are tenant export, retention, and deletion obligations?
10. Which components must scale independently?
11. How are shared files stored and encrypted?
12. What support access can a customer consent to?

## 17. Common failure modes to avoid

- Trusting `X-Tenant-Id` from the caller.
- Adding `TenantId` to primary tables but omitting it from joins, history, jobs, or audit tables.
- Letting platform tokens call tenant APIs directly.
- Treating an exit code of zero as verified success.
- Dispatching remote work directly inside the HTTP request.
- Retrying non-idempotent work after an ambiguous disconnect.
- Storing plaintext enrollment tokens, license keys, or agent credentials.
- Applying plan catalog changes retroactively without versioning.
- Processing billing webhooks without signature, deduplication, and ordering checks.
- Depending on frontend guards for authorization.
- Using SQLite-only tests for PostgreSQL production queries.
- Storing uploaded content inside replaceable build output.
- Assuming seed users or fixed test tenants exist.
- Returning generic HTTP 500 for known uniqueness or lifecycle conflicts.
- Allowing logging infrastructure failures to crash business requests.

## 18. Minimal reference checklist

A production-ready implementation should be able to demonstrate:

- [ ] tenant context comes only from validated identity;
- [ ] platform and tenant routes are mutually isolated;
- [ ] all tenant-owned rows and joins enforce tenant scope;
- [ ] PostgreSQL RLS blocks cross-tenant reads and writes;
- [ ] tenant provisioning is atomic;
- [ ] lifecycle transitions are constrained and audited;
- [ ] plans are versioned and entitlements are snapshotted;
- [ ] webhooks are signed, deduplicated, ordered, and transactional;
- [ ] durable operations survive API and worker restarts;
- [ ] ambiguous mutations become `OutcomeUnknown`, not automatic retries;
- [ ] agent credentials are hashed, rotatable, and revocable;
- [ ] remote results contain bounded logs and before/after verification;
- [ ] support access is short-lived and dual-actor audited;
- [ ] content is immutable, hash-verified, and stored durably;
- [ ] the application can run behind multiple API instances;
- [ ] full tests run against PostgreSQL as well as the local test store;
- [ ] OpenAPI generation, health, readiness, backup, restore, and offboarding are tested.

---

This blueprint preserves the useful SaaS methods implemented in TKM X Control while keeping the domain replaceable. The central reusable idea is that tenancy, identity, entitlements, durable work, evidence, and audit are platform primitives; individual product features are built on top of those primitives rather than implementing their own security and execution models.
