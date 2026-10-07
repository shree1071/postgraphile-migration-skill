# Intern Training Guide: Modernizing Backend Services to PostGraphile v5

Comprehensive guide and operational cheatsheet for interns migrating legacy REST `database-service` calls to direct PostGraphile v5 GraphQL operations in Zarthi microservices.

---

## 1. Quick Start: Using Antigravity Skill

The repository includes a dedicated Antigravity skill: [`.agents/skills/postgraphile-migration/`](.agents/skills/postgraphile-migration/).

When assigned an API to migrate, paste this prompt directly into Antigravity:

```text
Migrate this endpoint using /postgraphile-migration:
Target API: [PASTE METHOD OR ENDPOINT HERE, e.g. POST /module/actionable/update-description/v0.0.2]
Parent Branch: [PASTE PARENT BRANCH HERE, e.g. modernization/task-189/db-migration-objectives-requestmoduleactionableservice]
```

The skill guides the assistant through schema mapping, GraphQL document reuse, Jackson DTO creation, invariant enforcement, unit tests, and compliant branch routing.

---

## 2. Architecture Shift

```
[Legacy Flow]
Client -> Microservice -> GenericWebClientService -> database-service (8088) -> JPA / Hibernate -> PostgreSQL RDS

[Modern Flow]
Client -> Microservice -> PostgraphileHelper -> PostgraphileClientService -> PostGraphile v5 (Port 4000) -> PostgreSQL RDS
```

### Key Principles
1. **Zero calls to `database-service` (port 8088)**: The migrated endpoint must never invoke `genericWebClientService` or route through port 8088.
2. **Strict Direct Table Access**: Interact directly with underlying database tables (`requests`, `requests_services`, `actionables`, `actionables_steps`, etc.). Never query or mutate database views.

---

## 3. Step-by-Step Migration Lifecycle

### Step 1: Inspect Legacy Code and Map Schema
1. Find the endpoint in the controller (e.g. `RequestsModuleActionableV1ApiController.java`).
2. Trace the implementation in the legacy service (e.g. `RequestModuleActionableService.java`).
3. Note every table, column, and foreign key touched.
4. Verify field names and data types in `postgraphile-v5/schema.graphql`.

### Step 2: Prepare GraphQL Documents
Check `src/main/resources/graphql-documents/` first before creating new queries or mutations.
- **Reusable Documents**:
  - `create-actionable-with-step.graphql`: Creating new actionable with step 1.
  - `update-actionable-step-status.graphql`: Submitting or completing steps.
  - `update-actionable-status.graphql`: Discarding, cancelling, or completing actionables.
  - `find-actionable-by-id.graphql`: Fetching actionable details.
- **New Documents**:
  - Name files using action-oriented kebab-case: `<verb>-<entity>-<action>.graphql` (e.g. `update-request-owner.graphql`, `create-requests-service.graphql`).
  - Declare custom scalars explicitly: `scalar RecordId`, `scalar Dropdown`.
  - Check non-null constraints (e.g. `status: SingleLineText!` on `RequestInput`, default to `"Open"`).

### Step 3: Create Strongly-Typed DTOs
Under `src/main/java/com/zarthi/<module>/dto/graphql/`:
- Annotate with:
  ```java
  @Data
  @Builder
  @NoArgsConstructor
  @AllArgsConstructor
  @JsonIgnoreProperties(ignoreUnknown = true)
  ```
- Always include `@Builder` on inner payload classes so test fixtures can be created cleanly.

### Step 4: Implement Service Method & Controller Wiring
1. In `*GraphqlService.java`:
   - Validate user ID header: `requestInterceptorService.getUserIdHeader()`.
   - Execute via helper:
     ```java
     PostgraphileHelper.executeGraphQl(
         postgraphileClientService,
         requestInterceptorService,
         appId,
         DOC_NAME,
         variables,
         ResponseClass.class
     );
     ```
2. In the Controller:
   - Route endpoint directly to `actionableGraphqlService.<methodName>(...)`.

### Step 5: Test Verification
1. Run syntax and parsing tests on all GraphQL documents via `ExecutableOnlyDocumentSource`.
2. Write unit tests in `*GraphqlServiceEndpointsTest.java` verifying each endpoint method end-to-end with Jackson-deserialized fixtures.
3. Run Maven tests from `backend/`:
   ```bash
   mvn test -pl services/<module>
   ```

---

## 4. Non-Negotiable Database Invariants

Refer to [`.agents/skills/postgraphile-migration/INVARIANTS.md`](.agents/skills/postgraphile-migration/INVARIANTS.md) for full details.

1. **Check Constraint `chk_at_most_one_assignment`**:
   - On table `actionables_steps`: populate either `refUsersInRecordIdOwner` OR `refRequestsInRecordIdAssignedTo`, never both.
2. **Step Status Lifecycle**:
   - Step creation requires `status: "Planned"`.
   - For immediate execution (`INSTANT`), set actionable to `"Completed"` and step to `"Complete"`.
   - For draft execution (`DRAFT`), set actionable to `"Drafted"` and step to `"Draft"`.
   - For deferred status transitions upon creation, use `deferredPostCreatePatch: { status: "Draft" | "Complete" }`.
3. **Metadata Segregation**:
   - Draft form fields belong in `draftedData`.
   - Active and completed form fields belong in `metadata`.
4. **PostGraphile Scalar Types**:
   - `requestDependency`: scalar `Dropdown!` (not `SingleLineText`).
   - `status`: non-null `SingleLineText!` (default `"Open"`).
   - Deletion arrays: `[BigInt!]` (e.g. `requestsCollaboratorsByTheirRefRequestsInRecordIdDeleteList`).

---

## 5. Dev Database Verification and Cleanup Protocol

When testing queries manually in Ruru (GraphiQL) or DBeaver:

### Setting PostgreSQL Session Context
PostgreSQL has Row-Level Security (RLS) enabled. Set the session context before executing SQL:
```sql
SET app.CURRENT_APP_ID = '9f4c8a7b6e2d1c3a5b8f0d9e7c6a4b2f1e3d5c7a9b0e2f4c6d8a1b3c5e7f9d0';
SET app.CURRENT_USER_ID = '<your-user-uuid>';
-- If testing CDS:
SET app.is_cds = 'true';
```

### Verifying Rows in SQL
```sql
SELECT in_record_id, owner, summary, status, ref_requests_in_record_id_immediate_parent 
FROM requests WHERE in_record_id = <request_id>;

SELECT in_record_id, actionable_name, actionable_status, action_metadata 
FROM actionables WHERE request_subject = <request_id> ORDER BY in_record_id DESC;

SELECT in_record_id, step_no, status, ref_users_in_record_id_owner, ref_requests_in_record_id_assigned_to 
FROM actionables_steps WHERE ref_actionables_in_record_id = <actionable_id>;
```

### Dev Database Cleanup (Crucial)
Mutations executed in Ruru persist actual rows in `zarthi_dev`. Clean up test records after verification to prevent polluting the development database:
```sql
DELETE FROM requests_services WHERE ref_requests_record_id = <test_request_id>;
DELETE FROM requests_staffing WHERE ref_requests_record_id = <test_request_id>;
DELETE FROM actionables_steps WHERE ref_actionables_in_record_id IN (
    SELECT in_record_id FROM actionables WHERE request_subject = <test_request_id>
);
DELETE FROM actionables WHERE request_subject = <test_request_id>;
DELETE FROM requests WHERE in_record_id = <test_request_id>;
```

---

## 6. Git Sub-Branching & PR Routing Rules

Refer to [`.agents/skills/postgraphile-migration/BRANCHING.md`](.agents/skills/postgraphile-migration/BRANCHING.md) for full details.

### The Double-Dash Rule
In this repository, sub-branches MUST follow this exact pattern:
```text
<parent-feature-branch>--<intern-name>
```

- **Correct**:
  `modernization/task-189/db-migration-objectives-requestmoduleactionableservice--shreeharsha-phase-2`
- **Incorrect**:
  `modernization/task-189/db-migration-objectives-requestmoduleactionableservice-phase-2--shreeharsha`

### Why This Matters
The repository CI script (`pre-merge-checks/pr-branch-strategy.sh`) extracts the parent branch using:
```bash
PARENT_FEATURE=$(echo "$SOURCE" | sed 's/--[^-].*$//')
```
If you place any text like `-phase-2` before the `--`, the CI check fails with:
`REASON: Sub-branch must target its parent, not ...`
and automatically blocks the PR and sends a Teams alert.

### PR Creation Checklist
1. Create and checkout sub-branch from the assigned parent branch:
   ```bash
   git checkout -b <parent-feature-branch>--<intern-name>
   ```
2. Commit changes using Conventional Commits:
   - `feat(<module>): migrate <endpoint-name> to PostGraphile GraphQL`
   - `test(<module>): add unit tests for <endpoint-name>`
3. Push to origin:
   ```bash
   git push -u origin <parent-feature-branch>--<intern-name>
   ```
4. Create PR targeting the parent branch:
   ```bash
   gh pr create --base <parent-feature-branch> --head <parent-feature-branch>--<intern-name> ...
   ```
5. Confirm CI check `PR Validation - Branch Strategy Enforcement` passes.
