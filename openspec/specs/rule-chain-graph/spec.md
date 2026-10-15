# rule-chain-graph Specification

## Purpose
A rule chain is a directed graph of rule nodes that the ThingsBoard rule engine executes for a tenant. This spec records how the platform validates that graph today, which HTTP endpoints read and write it, who is allowed to call them, and what a save sets off (version control auto-commit, cluster broadcast, audit log, and a cascading rewrite of related rule chains). It is the behaviour a new graph-shape check — for example an "unreachable node" warning — has to fit into.

## Requirements

### Requirement: Rule chain entity validation rejects invalid chains by throwing
The system SHALL validate a `RuleChain` entity on every save through `RuleChainDataValidator`, and SHALL reject an invalid chain by throwing `DataValidationException`, so that no part of the save takes effect.

Validated on create: the tenant's rule-chain entity-count limit (`validateNumberOfEntitiesPerTenant`). Validated on create and update: the name is non-blank; a null `type` is defaulted to `CORE` rather than rejected; the chain is assigned to a tenant and that tenant exists; and at most one root chain exists per tenant, enforced separately for `CORE` (`getRootTenantRuleChain`) and for `EDGE` (`getEdgeTemplateRootRuleChain`).

#### Scenario: Chain without a name
- **WHEN** a rule chain with a blank name is saved
- **THEN** `DataValidationException` is thrown and nothing is persisted

#### Scenario: Chain with no type
- **WHEN** a rule chain whose `type` is null is saved
- **THEN** the validator sets `type` to `CORE` and the save proceeds

#### Scenario: Second root chain for a tenant
- **WHEN** a chain is saved with `root = true` and `type = CORE` while another `CORE` root chain with a different id exists for the tenant
- **THEN** `DataValidationException` "Another root rule chain is present in scope of current tenant!" is thrown

### Requirement: Graph metadata validation runs inside the save and rejects by throwing
The system SHALL validate rule chain metadata inside `BaseRuleChainService.saveRuleChainMetaData` before writing anything, and SHALL signal every failure by throwing `DataValidationException`, aborting the transaction.

Two checks run, in this order:
1. `RuleChainDataValidator.validateMetaDataFieldsAndConnections` — bean-constraint validation of the `RuleChainMetaData` object (`ConstraintValidator.validateFields`), then `validateCircles` over the connection list when it is non-empty.
2. `RuleChainDataValidator.validateRuleNode` per node, inside the node-persisting loop: bean-constraint validation of the node (`@Length` on `type`, `@NoXss` + `@Length` on `name`), then resolution of the node's config class from the `configClazz` property of its `@RuleNode` annotation by reflection and bean-constraint validation of the deserialised configuration.

`validateRuleNode` returns a `Throwable` instead of throwing when it fails to resolve or deserialise the configuration, but the save path ignores that return value, so an unresolvable node type does not fail the save — only the bean-constraint violations inside it do, because those throw.

#### Scenario: Node name containing markup
- **WHEN** metadata is saved whose node name violates `@NoXss`
- **THEN** `validateRuleNode` throws `DataValidationException` from within the save and the whole metadata save is rolled back

#### Scenario: Node type that cannot be resolved
- **WHEN** metadata is saved containing a node whose `type` has no resolvable `@RuleNode` annotation
- **THEN** `validateRuleNode` logs a warning and returns the `Throwable`, the save path discards it, and the save succeeds

### Requirement: The only graph-shape check today is a self-edge and cycle check
The system SHALL reject a connection whose `fromIndex` equals its `toIndex`, and SHALL reject a cycle that returns to the node the walk started from, by throwing `DataValidationException`. This is the only check that looks at the shape of the graph rather than at individual fields.

`validateCircles` builds a `Map<Integer, Set<Integer>>` from `fromIndex` to `toIndex`, ignoring the connection label (`type`), then walks it from every key. The walk carries no visited set and compares only against its own start node, so it is neither complete nor bounded: a cycle that the start node reaches but that does not return to it is not reported as a cycle, and the recursion does not terminate.

#### Scenario: Connection from a node to itself
- **WHEN** metadata is saved with a connection whose `fromIndex` equals its `toIndex`
- **THEN** `DataValidationException` "Can't create the relation to yourself." is thrown

#### Scenario: Cycle through the start node
- **WHEN** metadata is saved with connections 0 -> 1 and 1 -> 0
- **THEN** `DataValidationException` "Can't create circling relations in rule chain." is thrown

#### Scenario: Cycle the start node reaches but is not part of
- **WHEN** the connection map is 0 -> 1, 1 -> 2, 2 -> 1 and the walk starts at key 0
- **THEN** the recursion never terminates and a `StackOverflowError` is raised instead of `DataValidationException` (confirmed by running the algorithm in isolation over this map)

### Requirement: There is no reachability or unreachable-node check anywhere
The system SHALL NOT today check whether every node is reachable from the chain's first node, on any layer. `firstNodeIndex` is used only to resolve `RuleChain.firstRuleNodeId` when the metadata is saved and to recompute the index when the metadata is loaded; it is never compared against the connection graph. The Angular editor's `validate()` only folds a per-node `error` flag into `isInvalid` and refreshes node tooltips, with no graph traversal.

#### Scenario: A node no connection reaches
- **WHEN** metadata is saved whose node list contains a node that no connection points to and that is not `firstNodeIndex`
- **THEN** the save succeeds with no error and no warning, and the node is persisted with only its `CONTAINS` relation to the chain

### Requirement: Rule chain metadata is saved through one HTTP endpoint with full-replace semantics
The system SHALL expose `POST /api/ruleChain/metadata` (`RuleChainController.saveRuleChainMetaData`) as the endpoint that writes a rule chain's graph. It SHALL take the whole `RuleChainMetaData` as the request body and an optional `updateRelated` query parameter defaulting to `true`, SHALL clear the tenant's debug rate limits when `debugPerTenantEnabled` is set, SHALL authorise with `checkRuleChain(ruleChainId, Operation.WRITE)`, and SHALL return the reloaded metadata.

The graph itself is identified positionally: `connections` and `firstNodeIndex` are indices into the `nodes` list, and every save replaces the chain's node set and relations rather than patching them. `GET /api/ruleChain/{ruleChainId}/metadata` reads it back, recomputing the indices from the nodes sorted by created time then id.

#### Scenario: Save with a stale version
- **WHEN** metadata is posted whose `version` is non-null and differs from the stored chain's version
- **THEN** `EntityVersionMismatchException` is thrown and nothing is written

#### Scenario: Save for a chain that does not exist
- **WHEN** metadata is posted for a rule chain id that cannot be found
- **THEN** the DAO returns a failed `RuleChainUpdateResult` and the service turns it into a not-found error before any side effect runs

### Requirement: Other write routes reach the graph without the metadata endpoint
The system SHALL also persist rule chain graphs through `POST /api/ruleChains/import` (`importTenantRuleChains`), which assigns new random chain and node ids, rewrites tenant ids inside node configurations, saves each chain, and then calls `saveRuleChainMetaData` once per chain through the service's own proxy so each metadata save gets its own transaction. A failure on one chain is recorded in that chain's `RuleChainImportResult` and does not abort the import.

Import authorises differently from the metadata endpoint: it calls `accessControlService.checkPermission(user, Resource.RULE_CHAIN, Operation.WRITE)` directly and calls the DAO service, bypassing `TbRuleChainService` and therefore all of the side effects below. The other rule-chain write routes — `POST /api/ruleChain`, `POST /api/ruleChain/device/default`, `POST /api/ruleChain/{id}/root`, `POST /api/ruleChain/{id}/edgeTemplateRoot`, `POST`/`DELETE /api/ruleChain/{id}/autoAssignToEdge`, and `POST`/`DELETE /api/edge/{edgeId}/ruleChain/{ruleChainId}` — change the chain entity or its assignments and never the node graph.

#### Scenario: Import of several chains where one is invalid
- **WHEN** a `RuleChainData` payload is imported in which one chain's metadata fails validation
- **THEN** that chain's metadata save rolls back on its own and the remaining chains are still imported

### Requirement: Every rule chain endpoint is restricted to tenant administrators and then to the entity
The system SHALL guard every endpoint in `RuleChainController`, read and write alike, with `@PreAuthorize("hasAuthority('TENANT_ADMIN')")`, so a customer user has no access to this area at all. It SHALL then authorise the specific entity through `BaseController.checkRuleChain(ruleChainId, operation)`, which resolves the chain and delegates to `accessControlService` — `Operation.READ` on the getters, `Operation.WRITE` on the metadata save and the root/assignment changes.

`Resource.RULE_CHAIN` maps to the tenant standalone-entity checker in `TenantAdminPermissions` and to the system checker in `SysAdminPermissions`; `DefaultUserPermissionsService` grants a system administrator `Operation.ALL`. A new read-only endpoint in this area needs no new `Resource`: it reuses `checkRuleChain(id, Operation.READ)`.

#### Scenario: Customer user calls the metadata endpoint
- **WHEN** a user with the `CUSTOMER_USER` authority calls `POST /api/ruleChain/metadata`
- **THEN** Spring Security rejects the request before any controller code runs

#### Scenario: Tenant administrator calls it for another tenant's chain
- **WHEN** a tenant administrator posts metadata for a rule chain belonging to a different tenant
- **THEN** `checkRuleChain(..., Operation.WRITE)` denies the request

### Requirement: A metadata save sets off four side effects in a fixed order
The system SHALL run the following after `DefaultTbRuleChainService.saveRuleChainMetaData` persists the graph, in this order: cascading rewrite of related chains, version-control auto-commit, reload of the saved metadata, cluster broadcast, audit log.

1. **Cascading rewrite.** When `updateRelated` is true, `updateRelatedRuleChains` compares old and new names of this chain's output nodes and renames the matching input labels in *other* chains. A label renamed to two different names is dropped from the rename map and logged as a conflict, as is a renamed label still present in the chain.
2. **Auto-commit.** `autoCommit` is called for this chain alone, or for the whole set of changed `RULE_CHAIN` uuids when related chains were rewritten.
3. **Cluster broadcast.** `tbClusterService.broadcastEntityStateChangeEvent(..., ComponentLifecycleEvent.UPDATED)` is sent for each *related* chain that was rewritten, and only when this chain's type is `CORE`; this is what makes the rule engine actors reinitialise. The saved chain itself is announced by the DAO layer, which publishes a `SaveEntityEvent` for it at the end of the transaction.
4. **Audit log.** `logEntityAction(..., ActionType.UPDATED, ...)` for this chain, then for each rewritten related chain whose metadata could be reloaded (`CORE` only).

#### Scenario: Renaming an output node of a chain other chains link to
- **WHEN** a chain's output node is renamed from `Success` to `Done` and `updateRelated` is true
- **THEN** the chains whose input labels referenced `Success` are rewritten, auto-committed with this chain, broadcast as `UPDATED`, and each gets its own audit entry

#### Scenario: Save fails
- **WHEN** any step of the metadata save throws
- **THEN** an audit entry is written with the error and with `ActionType.ADDED` — not `UPDATED` — against an empty rule chain id, and the exception is rethrown

### Requirement: The metadata save is serialised per rule chain
The system SHALL run the graph write inside a single transaction, under a covering advisory lock keyed on the rule chain id (`relationWriteLock.withCoveringLock`), which serialises concurrent edits of the same chain on Citus and is a no-op on plain PostgreSQL. The `rule_chain` row write is deliberately placed inside that block and after the relation writes, so the covering lock is taken before the row. Cross-chain `USES` relation rows written for rule-chain-input nodes are an accepted exception: they are written under this chain's key only.

#### Scenario: Two administrators save the same chain at once
- **WHEN** two metadata saves for one rule chain run concurrently on Citus
- **THEN** the covering advisory lock serialises them, and the loser fails the version check with `EntityVersionMismatchException`

## Code map
- `dao/src/main/java/org/thingsboard/server/dao/service/validator/RuleChainDataValidator.java`: all rule chain validation — entity checks that throw, metadata/node checks, and `validateCircles`, the only graph-shape rule.
- `dao/src/main/java/org/thingsboard/server/dao/service/ConstraintValidator.java`: static bean-constraint validation that turns violations into `DataValidationException`.
- `dao/src/main/java/org/thingsboard/server/dao/rule/BaseRuleChainService.java`: the transactional graph write (`saveRuleChainMetaData`, `doSaveRuleChainMetaData`), the index-to-relation mapping, the load path, and `importTenantRuleChains`.
- `application/src/main/java/org/thingsboard/server/service/rule/DefaultTbRuleChainService.java`: the save's four side effects and the output-label rewrite of related chains.
- `application/src/main/java/org/thingsboard/server/service/rule/TbRuleChainService.java`: the interface those side effects are declared on.
- `application/src/main/java/org/thingsboard/server/controller/RuleChainController.java`: the HTTP surface, including the metadata save, the metadata read, the output-label endpoints and the import route.
- `application/src/main/java/org/thingsboard/server/controller/BaseController.java`: `checkRuleChain`, the per-entity authorisation gate.
- `application/src/main/java/org/thingsboard/server/service/security/permission/`: `Resource.RULE_CHAIN` permission mapping for tenant and system administrators.
- `common/data/src/main/java/org/thingsboard/server/common/data/rule/RuleChainMetaData.java`, `NodeConnectionInfo.java`, `RuleNode.java`: the transport shape — index-based connections and the node constraints that are validated.
- `ui-ngx/src/app/modules/home/pages/rulechain/rulechain-page.component.ts`: the editor's `validate()` (per-node error flags and tooltips only) and `saveRuleChain()`, which builds the metadata payload.
- `dao/src/test/java/org/thingsboard/server/dao/service/RuleChainServiceTest.java`, `CitusRuleChainServiceTest.java`: the DAO-level save and concurrency coverage. There is no dedicated test class for `RuleChainDataValidator`.
- `application/src/test/java/org/thingsboard/server/service/install/InstallScriptsTest.java`: the only caller of `validateMetaData`, which it uses to assert the bundled rule chain templates validate.
- `application/src/test/java/org/thingsboard/server/rules/flow/AbstractRuleEngineFlowIntegrationTest.java`, `application/src/test/java/org/thingsboard/server/controller/AbstractRuleEngineControllerTest.java`: end-to-end coverage of saving a graph and running messages through it.

Source: commit 97b78f4989902ad8448d1dca573063fc4a5d40d4
