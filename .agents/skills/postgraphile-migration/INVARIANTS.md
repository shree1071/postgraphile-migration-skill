# PostGraphile Migration Database Invariants

Core invariants and schema rules when migrating legacy REST database-service calls to direct PostGraphile v5 GraphQL operations.

---

## 1. Table Access Rule
- Target underlying database tables directly (e.g. `requests`, `requests_collaborators`, `requests_services`, `requests_staffing`, `actionables`, `actionables_steps`, `users`).
- Never query or mutate database views during migrations. Views do not support direct GraphQL mutations and often lack required mutation triggers.
- Eliminate all HTTP calls to `database-service` (port 8088) and `genericWebClientService`.

---

## 2. Check Constraint: chk_at_most_one_assignment
- Table: `actionables_steps`
- Rule: A step can be assigned to either a user OR a request, never both simultaneously.
- Fields:
  - `refUsersInRecordIdOwner`: User ID assigned to step.
  - `refRequestsInRecordIdAssignedTo`: Request ID assigned to step.
- Enforcement: When assigning a step, populate exactly one of these two fields and leave the other null.

---

## 3. Actionable and Step Lifecycle
- Actionable Statuses: `Drafted`, `Planned`, `Completed`, `Discarded`.
- Step Statuses: `Draft`, `Planned`, `Complete`, `Discard`.
- Step Creation Invariant:
  - Step creation defaults to `status: "Planned"`.
  - Immediate execution: set `actionableStatus: "Completed"` and patch step with `status: "Complete"`.
  - Draft creation: set `actionableStatus: "Drafted"` and patch step with `status: "Draft"`.
  - Planned creation: set `actionableStatus: "Planned"` and keep step as `status: "Planned"`.
- Deferred Post-Create Patch:
  - For non-planned statuses during initial creation, use `deferredPostCreatePatch: { status: "Complete" | "Draft" }` or invoke step update mutation immediately following creation.

---

## 4. Metadata Segregation
- Draft payloads: Store unsaved/in-progress form fields in the `draftedData` JSON object.
- Active / Executed payloads: Store committed configuration and fields in the `metadata` JSON object.
- Never write execution parameters directly into `draftedData` when `executionType` is `INSTANT`.

---

## 5. Schema Scalar Mapping
Verify all types against `postgraphile-v5/schema.graphql` before authoring documents:
- `RecordId!`: Identifiers on relational records. Pass as numeric ID or string ID.
- `Dropdown!`: PostGraphile dropdown scalar (e.g. `requestDependency`).
- `SingleLineText!`: Required text scalar (e.g. `status: SingleLineText!` on `RequestInput`, default to `"Open"`).
- `BigInt!`: Long integers for foreign keys and record identifiers.
- `Decimal1!`: Decimal numbers for hours or quantities (e.g. `estimatedQuantity`).
- Deletion lists: Relations requiring ID arrays for deletion use `[BigInt!]` (e.g. `requestsCollaboratorsByTheirRefRequestsInRecordIdDeleteList`).

---

## 6. Database Verification & Session Context
When verifying mutations directly in PostgreSQL or diagnosing Ruru execution:

### Session Context Setup
PostgreSQL RLS and audit triggers require session variables to be set within the session or transaction:
```sql
SET app.CURRENT_APP_ID = '<objective_app_id>';
SET app.CURRENT_USER_ID = '<user_uuid_or_numeric_id>';
-- For CDS operations:
SET app.is_cds = 'true';
```

### Table Verification Queries
1. **Requests Base Table**:
   ```sql
   SELECT in_record_id, owner, summary, status, ref_requests_in_record_id_immediate_parent, in_created_time
   FROM requests
   WHERE in_record_id = <request_id>;
   ```
2. **Actionables & Steps**:
   ```sql
   SELECT in_record_id, actionable_name, actionable_status, created_by, action_metadata
   FROM actionables
   WHERE request_subject = <request_id>
   ORDER BY in_record_id DESC;

   SELECT in_record_id, step_no, status, ref_users_in_record_id_owner, ref_requests_in_record_id_assigned_to
   FROM actionables_steps
   WHERE ref_actionables_in_record_id = <actionable_id>;
   ```
3. **Extension Tables**:
   ```sql
   -- Services
   SELECT * FROM requests_services WHERE ref_requests_record_id = <request_id>;
   -- Staffing
   SELECT * FROM requests_staffing WHERE ref_requests_record_id = <request_id>;
   -- Collaborators
   SELECT * FROM requests_collaborators WHERE ref_requests_in_record_id = <request_id>;
   ```

### Test Data Cleanup Protocol
When running manual tests in `zarthi_dev` via Ruru:
- Mutations persist actual rows in PostgreSQL.
- Check generated `in_record_id` values from GraphQL responses.
- Clean up test records after verification to prevent polluting shared development environments:
  ```sql
  DELETE FROM requests_services WHERE ref_requests_record_id = <test_request_id>;
  DELETE FROM requests_staffing WHERE ref_requests_record_id = <test_request_id>;
  DELETE FROM actionables_steps WHERE ref_actionables_in_record_id IN (SELECT in_record_id FROM actionables WHERE request_subject = <test_request_id>);
  DELETE FROM actionables WHERE request_subject = <test_request_id>;
  DELETE FROM requests WHERE in_record_id = <test_request_id>;
  ```
