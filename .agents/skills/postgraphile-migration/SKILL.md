---
name: postgraphile-migration
description: Migrate legacy database-service calls to PostGraphile v5 GraphQL operations. Use when asked to migrate endpoints to PostGraphile, modernise database calls, or replace database-service REST calls with GraphQL.
---

# PostGraphile Migration

Execute the end-to-end migration of legacy REST `database-service` (port 8088) calls to direct PostGraphile v5 GraphQL operations.

---

## 1. Audit Legacy Endpoint and Map Schema

1. Locate the endpoint in the relevant Controller (e.g. `RequestsModuleActionableV1ApiController.java`).
2. Trace the execution path into the legacy Service (e.g. `RequestModuleActionableService.java`).
3. Audit every database entity, field, and request/response payload touched.
4. Map all entities directly to underlying PostgreSQL database tables.
   - Reference: See [INVARIANTS.md](INVARIANTS.md) for table access rules.
5. Inspect `postgraphile-v5/schema.graphql` or open the interactive Docs explorer in Ruru (`http://localhost:4000/ruru` or `https://api.dev.zarthi.com/graphql`) to verify field names, scalar types, and required non-null constraints for all mapped tables.

**Completion Criterion**: Produce an exhaustive field-to-table mapping for the target endpoint. Every field maps strictly to an underlying table column, with zero references to database views.

---

## 2. Prepare GraphQL Documents

1. Check `src/main/resources/graphql-documents/` for existing operations to reuse:
   - Actionable creation: `create-actionable-with-step.graphql`
   - Step updates: `update-actionable-step-status.graphql`
   - Actionable status updates: `update-actionable-status.graphql`
   - Entity lookups: `find-actionable-by-id.graphql`, `find-request-by-in-record-id.graphql`
2. Create new documents only when an operation cannot be fulfilled by an existing parameterized query or mutation.
3. Name new files using action-oriented kebab-case: `<verb>-<entity>-<action>.graphql` (e.g. `update-request-owner.graphql`, `create-requests-service.graphql`).
4. Align variable types and payload selections with `postgraphile-v5/schema.graphql`.
   - Verify non-null constraints (e.g. `status: SingleLineText!`, `requestDependency: Dropdown!`).
   - Declare custom scalars explicitly in the document where required (`scalar RecordId`, `scalar Dropdown`).

**Completion Criterion**: Every needed GraphQL document exists under `src/main/resources/graphql-documents/` and parses validly against the PostGraphile schema with exact scalar matching.

---

## 3. Implement Strongly-Typed DTOs

1. Create request and response DTO classes in the service package: `dto/graphql/model/<domain>/`.
2. Annotate each class with:
   ```java
   @Data
   @Builder
   @NoArgsConstructor
   @AllArgsConstructor
   @JsonIgnoreProperties(ignoreUnknown = true)
   ```
3. Include `@Builder` on inner payload and node classes to support test fixture instantiation.
4. Mirror the PostGraphile response hierarchy:
   - Root Field -> Payload / Connection -> Node

**Completion Criterion**: Every GraphQL variable set and response payload maps to a compiled DTO class containing complete Jackson annotations and builder methods.

---

## 4. Implement GraphQL Service Layer

1. Add the migration method to the designated GraphQL service (e.g. `RequestModuleActionableGraphqlService.java`).
2. Validate user identity via `requestInterceptorService.getUserIdHeader()`. Reject missing headers with `HttpStatus.UNAUTHORIZED`.
3. Execute GraphQL queries and mutations via `PostgraphileHelper.executeGraphQl`:
   ```java
   PostgraphileHelper.executeGraphQl(
       postgraphileClientService,
       requestInterceptorService,
       objectiveAppId,
       DOC_NAME,
       variables,
       ResponseGqlDTO.class
   );
   ```
4. Enforce domain invariants:
   - Check constraint `chk_at_most_one_assignment`: Populate either user owner or request assignee on `actionables_steps`, never both.
   - Lifecycle state: Steps initialize with `status: "Planned"`; apply `deferredPostCreatePatch` or explicit step status update for `Draft` and `Completed` executions.
   - Metadata structure: Place draft form fields under `draftedData`; place executed fields under `metadata`.
   - Reference: See [INVARIANTS.md](INVARIANTS.md) for full constraint details.
5. Update the Controller to route the API directly to the GraphQL service method.

**Completion Criterion**: The endpoint routes completely through the GraphQL service, produces matching legacy response contracts, and performs zero calls to `database-service` (port 8088) or `genericWebClientService`.

---

## 5. Verify Test Suite and Database State

1. Document Parsing Tests:
   - In `ActionableGraphQLDocumentsTest.java`, parse all `.graphql` files through `ExecutableOnlyDocumentSource` to verify schema validity.
   - Test JSON deserialization of sample GraphQL responses into DTOs.
2. Endpoint Execution Tests:
   - In `*GraphqlServiceEndpointsTest.java`, test each migrated endpoint method end-to-end.
   - Mock `postgraphileClientService.executeQueryByDocumentName` returning deserialized JSON fixtures:
     ```java
     objectMapper.readValue(fixtureJson, ResponseGqlDTO.class)
     ```
   - Assert returned `GenericMessageResponse` or DTO fields match legacy specifications.
   - Remove unused stubbings to maintain strict Mockito compliance.
3. Direct Database Row Verification (SQL / Ruru):
   - When executing manual smoke tests via Ruru / GraphiQL or direct DB client, set PostgreSQL session context:
     ```sql
     SET app.CURRENT_APP_ID = '<app_id>';
     SET app.CURRENT_USER_ID = '<user_uuid_or_id>';
     ```
   - Query the underlying tables directly in SQL to verify rows created or updated:
     - Check `requests`: Verify `in_record_id`, `owner`, `summary`, `status`, `ref_requests_in_record_id_immediate_parent`.
     - Check `actionables`: Verify `actionable_status`, `created_by`, `action_metadata`.
     - Check `actionables_steps`: Verify `status`, `step_no`, `ref_users_in_record_id_owner`, and `chk_at_most_one_assignment`.
     - Check extension tables (`requests_services`, `requests_staffing`, `requests_collaborators`): Verify linkage to request ID.
   - Test Data Cleanup in Dev DB:
     - If manual mutations were executed in `zarthi_dev` via Ruru, clean up generated test rows to prevent cluttering shared environments:
       Delete or cascade test rows by `in_record_id` created during the session.
4. Run the automated test suite:
   ```powershell
   mvn test -pl <service-directory>
   ```

**Completion Criterion**: All document parsing and endpoint unit tests pass with zero failures and zero errors (`BUILD SUCCESS`). Direct database queries confirm that mutations create or update rows in underlying tables with correct column values, respecting all triggers and check constraints, with any exploratory Ruru test rows cleaned up.

---

## 6. Branch Strategy and PR Routing

1. Verify sub-branch name matches repository double-dash convention:
   `<parent-feature-branch>--<sub-scope>`
   - Reference: See [BRANCHING.md](BRANCHING.md) for syntax rules and CI routing logic.
2. Commit changes with conventional commit messages:
   - `feat(<domain>): migrate <endpoints> to PostGraphile GraphQL`
   - `test(<domain>): add unit tests for migrated endpoints`
3. Push the sub-branch to `origin`.
4. Create the Pull Request targeting the exact `<parent-feature-branch>`.
5. Verify `PR Validation - Branch Strategy Enforcement` check passes in GitHub Actions.

**Completion Criterion**: Pull request created targeting the parent feature branch with all pre-merge branch routing checks green.
