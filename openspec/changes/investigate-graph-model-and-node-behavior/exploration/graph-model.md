# Graph model

Rule-chain nodes, connections, relation labels, the entry point and referenced chains.

What the ticket needs from this area: enough ground truth to **define what "unreachable" means** for this graph.

These are exploration notes, not a spec: no requirements, scenarios or deltas. No main spec exists for this domain yet (`openspec/specs/` is empty, which is normal — main specs appear when a change is archived).

## Current behaviour

### The graph exists in three representations, and they are not equivalent

**1. REST/JSON wire format — index-based (`RuleChainMetaData`)**

- A node is an element of `nodes: List<RuleNode>`; its identity on the wire is its **list position**. `RuleNode.id` absent ⇒ create, present ⇒ update.
- An edge is `NodeConnectionInfo{fromIndex, toIndex, type}` — three fields, no ids. `type` is the connection label, a free-form string.
- The entry point is `firstNodeIndex: Integer`, nullable.
- `ruleChainConnections: List<RuleChainConnectionInfo>{fromIndex, targetRuleChainId, type, additionalInfo}` is the legacy cross-chain edge (see below).
- Also carried: `notes` (canvas stickies) and `version` (optimistic lock).

**2. Persisted form — rows in the `relation` table plus one scalar column**

`relation`'s primary key is `(from_id, from_type, relation_type_group, relation_type, to_id, to_type)` (`dao/src/main/resources/sql/schema-entities.sql:526`). Three row kinds matter:

- **Membership**: `RULE_CHAIN -> RULE_NODE`, type `Contains`, group `RULE_CHAIN` (`BaseRuleChainService.java:294`). This is the only authoritative answer to "which nodes are in this chain" — `rule_node.rule_chain_id` exists as a column but enumeration goes through relations (`getRuleChainNodes` → `getRuleChainToNodeRelations`, `:473`, `:1007`).
- **Connection**: `RULE_NODE -> RULE_NODE`, `relation_type` = the label, group `RULE_NODE` (`:322-327`). **The label is part of the primary key**, so two labels between the same pair are two rows, and the same (from,to,label) twice collapses to one.
- **Cross-chain reference**: `RULE_CHAIN -> RULE_CHAIN`, type `Uses`, group `COMMON`, derived from each input node's `configuration.ruleChainId` (`getRuleChainInputRelation`, `:370-384`). This is bookkeeping for "which chains reference which", **not** a routing edge.

The entry point is `rule_chain.first_rule_node_id` (`schema-entities.sql:225`).

**3. Runtime actor routing**

`RuleChainActorMessageProcessor` holds `nodeActors: Map<RuleNodeId, RuleNodeCtx>` and `nodeRoutes: Map<RuleNodeId, List<RuleNodeRelation>>`, built in `initRoutes` (`:159-185`) directly from `service.getRuleNodeRelations(...)` — i.e. from `relation` group `RULE_NODE`, **never from `RuleChainMetaData`**. Indices do not exist here. The entry point is `firstNode = nodeActors.get(ruleChain.getFirstRuleNodeId())` (`:182`).

### What each conversion loses

- **Save** (`doSaveRuleChainMetaData`, `BaseRuleChainService.java:235-368`) maps indices to ids. It **deletes every outgoing relation of every existing node first** (`:256`) and then re-inserts: the persisted graph is a full replace, not a diff. Nodes absent from the payload are deleted (`:277-280`, `:309-311`).
- **Load** (`loadRuleChainMetaData`, `:387-426`) maps ids back to indices, assigning them by `sort by (createdTime, id)` (`:397`) — so **indices are renumbered on load and are not stable identifiers**. Connections are re-sorted by `(from,to,type)` (`:421`).
- Lost on a round trip: client-chosen node order; any `RULE_NODE -> RULE_CHAIN` relation row, which is **dropped with a `log.warn("Unsupported node relation")`** (`:415-417`) even though the runtime still routes on it (`:358-360`); and `ruleChainConnections`, which `loadRuleChainMetaData` never repopulates, so a GET always returns it null.

### The entry point: `firstNodeIndex` vs `firstRuleNodeId`

- The UI sets `firstNodeIndex` only if an edge exists from the synthetic input connector: `saveRuleChain()` (`rulechain-page.component.ts:1698-1701`) does `const firstNodeEdge = edges.find(e => e.source === inputConnectorId+''); if (firstNodeEdge) firstNodeIndex = nodes.indexOf(firstNode)`. With no such edge the field is **left undefined**, not -1.
- The server derives `firstRuleNodeId = firstNodeIndex == null ? null : nodes.get(firstNodeIndex).getId()` (`BaseRuleChainService.java:312-320`). A null index clears the column. But the whole block sits inside `if (nodes != null)` (`:313`), so a metadata POST with `nodes: null` leaves `first_rule_node_id` untouched.
- The two can diverge. If `first_rule_node_id` points at a node no longer `Contains`-related to the chain, `ruleNodeIndexMap.get(firstRuleNodeId)` returns null (`:404`), so `firstNodeIndex` comes back **absent** while the column is still set. The UI's load guard is `if (firstNodeIndex > -1)` (`rulechain-page.component.ts:653`), and `undefined > -1` is false, so the connector is drawn unconnected and the next save clears the column — the UI silently converges them.
- Nothing bounds-checks `firstNodeIndex`, `fromIndex` or `toIndex`; out of range gives a raw `IndexOutOfBoundsException` from `nodes.get(...)` (`:315`, `:323`).
- **At runtime with no entry point the chain is a silent black hole.** `initRoutes` sets `firstNode = null` and `state = ACTIVE` regardless (`:182-184`). `onTellNext(msg, true)` with a null `ruleNodeId` then takes `targetCtx = firstNode == null` and calls `msg.getCallback().onSuccess()` (`:214-220`) — the message is acked and dropped, at `log.trace` level, with no error and no metric. Same in `onRuleChainToRuleChainMsg` (`:266-269`).

### Cross-chain connections: the current mechanism, and the legacy one

The current mechanism is **ordinary nodes and ordinary edges**, not `RuleChainConnectionInfo`:

- `TbRuleChainInputNode` (`rule-engine/rule-engine-components/.../flow/TbRuleChainInputNode.java`) holds the target in `configuration.ruleChainId` and is annotated `relationTypes = {}`, `customRelations = true`, `ruleChainNode = true`. Its **outgoing labels are the target chain's output-node names** — exactly what `GET /api/ruleChain/{id}/output/labels` serves (`RuleChainController.java:169-176` → `DefaultTbRuleChainService.getRuleChainOutputLabels:64-72`, which returns the names of the target's `TbRuleChainOutputNode`s).
- `TbRuleChainOutputNode` is `outEnabled = false` and calls `ctx.output(msg, ctx.getSelf().getName())` — **the node's name is the label**.
- `DefaultTbRuleChainService.getOutputLabelUsage` (`:74-110`) is the reverse index: find every input node pointing at this chain and report the relation types on its outgoing edges. Used to rename links in *other* chains when an output node is renamed.

**Crossing a chain boundary lands on the target's entry point**, not on any node "inside" it. `TbRuleChainInputNode.onMsg` calls `ctx.input(msg, targetRuleChainId)`; `DefaultTbContext.input` (`:203-227`) pushes `(selfChainId, selfNodeId)` onto the message's processing stack, rebuilds the message with `ruleChainId = target` and `resetRuleNodeId()`, and re-enqueues. On consumption the target chain's actor sees empty relation types ⇒ `onTellNext(msg, true)` with a null rule-node id ⇒ `targetCtx = firstNode`. The return path: the output node's `ctx.output` (`:230-238`) pops the stack and sends `RuleChainOutputMsg(callerChainId, callerInputNodeId, label)`, so the **caller** resumes at the input node's outgoing edges matching that label. An empty stack just acks.

`RuleChainConnectionInfo` is legacy and no current client writes it: the UI save builds only `connections`, the loader never sets it, and the UI import actively migrates it away (`import-export.service.ts: processOldRuleChainConnections`). The `doSaveRuleChainMetaData` branch that handles it (`:329-361`) **synthesises a brand-new `TbRuleChainInputNode`** per entry and emits `RULE_CHAIN->newNode "Contains"` plus `fromNode->newNode <type>` — an on-the-fly upgrade to the input-node model. That synthesised node is not added to `ruleChainMetaData.nodes`, gets no `ruleNodeUpdater` / `validateRuleNode` / `setSingletonMode` treatment, and gets no `Uses` row.

### Validation today

All of it is `RuleChainDataValidator.validateMetaDataFieldsAndConnections` (`:85-90`), called once from `saveRuleChainMetaData` (`BaseRuleChainService.java:207`) after the version check. It does two things: bean-level `ConstraintValidator.validateFields` (annotations only, nothing graph-shaped), and — **only when `connections` is non-empty** — `validateCircles`.

`validateCircles` (`:108-131`) builds `Map<Integer, Set<Integer>>` from `fromIndex -> {toIndex}`, rejecting `from == to` with "Can't create the relation to yourself.", then for each key runs a DFS that throws "Can't create circling relations in rule chain." iff it reaches `to == from`. **Labels are discarded**, so two distinctly-labelled edges collapse.

The algorithm has **no visited set and no depth bound**. I replicated it standalone (`/workdir/probe/Probe.java`, run under JDK 25) and confirmed:

- `0->1, 1->2, 2->1` — a cycle that excludes the lowest-indexed node — dies with **`StackOverflowError`**, not `DataValidationException`. That is an `Error`, so it escapes the validator and the transaction as a 500 rather than a clean 400.
- `0->1, 1->0` reports the cycle correctly.
- 20 chained diamonds (60 nodes, fully acyclic) are accepted but take ~690 ms — cost is the number of distinct *paths*, not edges, so it is exponential on legitimate acyclic graphs.

Not covered anywhere: reachability or orphan detection; bounds on any index; whether the entry point is a sensible node; whether an edge's label is legal for the source node's `relationTypes`/`customRelations` (UI-only, via `rule-chain.service.ts: getRuleNodeSupportedLinks`); whether an edge out of an input node matches a real output label in the target; whether `inEnabled=false` nodes have incoming edges or `outEnabled=false` nodes have outgoing ones; cross-chain cycles (only the runtime stack check in `ctx.input` catches those). In the UI, `validate()` (`rulechain-page.component.ts:1647-1657`) only aggregates per-node `node.error` — there is **no graph-level validation in the UI to extend**.

### "unreachable" does not exist in this codebase today

A repo-wide search for `unreachable`, `reachab`, `orphan` and `disconnected` returns no hit concerning the rule-chain graph. The two nearest precedents are elsewhere:

- `dao/.../agent/StepLinkedListUtils.java:105-142` — orphan *and* cycle detection for agent compose-steps ("Orphaned steps detected"), with tests. A different graph, but the only orphan-detection precedent in the repo.
- `rulechain-page.component.ts:1047-1062` — `toIndexSet` → `noInputNodes` → `possibleInputNodes` filtered by `inEnabled`, used by "create nested rule chain from selection" to pick the extracted chain's entry node. In-degree reasoning, but not a validity check and it never warns.

### Sources of reachability a definition must account for

1. **The entry point** (`firstRuleNodeId`) — the only in-chain entry for queue-delivered messages, for `RuleChainToRuleChainMsg`, and for cross-chain `ctx.input`. May be null.
2. **Normal `connections` edges, regardless of label.** Runtime matching is `relationType.equalsIgnoreCase(...)` (`RuleChainActorMessageProcessor.contains:376-386`) — **case-insensitive at runtime but case-sensitive in the relation primary key**, so `"success"` and `"Success"` are two stored edges that both fire.
3. **Cross-chain entry from another chain**: any `TbRuleChainInputNode` anywhere in the tenant whose config points at this chain makes this chain's *entry point* reachable from outside. Statically discoverable via the `Uses` rows — **except** when `forwardMsgToDefaultRuleChain = true`, where the target is resolved from the originator's device/asset profile at runtime (`TbRuleChainInputNode.java:63-64`, `getOriginatorDefaultRuleChainId`) and so is unknowable statically.
4. **Root-chain and profile-default entry**: the tenant root chain's entry point receives every untargeted message, and any chain used as a device/asset profile `default_rule_chain_id` (`schema-entities.sql:305`, `:349`) or an edge's `root_rule_chain_id` is an entry too.
5. **Output nodes** are sinks within their chain but sources in the caller. An output node with no incoming edge inside its chain is dead; one with incoming edges makes the *caller's* input-node successors reachable. The hop is via the processing stack, which is dynamic.
6. **Legacy `RULE_NODE -> RULE_CHAIN` relation rows** still route at runtime (`:358-360`) but are invisible in `RuleChainMetaData` (`:415-417`). A traversal over the JSON sees less than one over `relation`.
7. **Re-injection into the queue**: `ctx.enqueue*` and the "To Root Rule Chain" relation type re-enter the *root* chain's entry point; `TbCheckpointNode` re-enqueues to another queue and re-enters via `Success`. Configuration-opaque hops.
8. **`ctx.tellSelf`** (`DefaultTbContext.java:196-199`) — a node can be live purely on its own timer.
9. **`inEnabled = false` nodes are entry points by construction** and can never have an incoming edge: `TbMsgGeneratorNode`, `TbAlarmsCountNode`, `TbAggLatestTelemetryNode` (confirmed: those are the only three in `rule-engine/`). The UI only draws a left connector when `inEnabled` (`rulechain-page.component.ts:630-637`). **They must never be called unreachable.**
10. `singletonMode` / `queueName` affect *where* a node executes, not whether it is reachable — they are not inputs to a reachability definition.
11. **`Failure` edges** are first-class with no special routing. The only asymmetry is at `relationsCount == 0`, where an unrouted `Failure` becomes a `RuleNodeException` instead of a success ack (`:299-311`).

### Ambiguities a definition of "unreachable" has to settle

- **Is a node reachable only via a `Failure` edge "reachable"?** It genuinely executes. Excluding `Failure` flags every error-handling subtree; including it loses most of the value for the common "forgot to wire it up" case.
- **Is an `inEnabled=false` generator with no outgoing edges unreachable, or merely useless?** It runs, but its output goes nowhere. Symmetrically for an `outEnabled=false` output node with no incoming edges.
- **What does an absent entry point mean?** With `firstNodeIndex == null`, every node is unreachable from inside, yet a chain driven entirely by generator nodes is a legitimate design.
- **Scope: per chain or per tenant?** Per-chain, an output node is a sink and an input node's successor labels depend on a *foreign* chain's node names. Per-tenant requires the full `Uses` closure plus profile defaults plus the dynamic `forwardMsgToDefaultRuleChain` targets, which cannot be resolved statically.
- **Label fidelity.** A node is only truly reachable via labels its source actually emits. `nodeRoutes` keeps every stored label (over-approximates: a `switch` whose script never returns a branch); consulting `@RuleNode(relationTypes)` under-approximates for `customRelations = true` nodes.
- **Case-differing duplicate labels** (point 2 above).
- **Which representation is authoritative?** A save-time validator sees only the JSON; a reporter reading `relation` rows sees the legacy cross-chain edges too.

**Why a naive "BFS from `firstNodeIndex`" is wrong**: it always flags `inEnabled=false` generators; it flags everything when the entry point is absent, including generator-driven chains; it misses re-entry via `enqueue`/checkpoint/"To Root Rule Chain"/`tellSelf`; it sees no cross-chain structure, so it cannot tell that output node `"X"` is dead because no caller links label `"X"`; and it reports **indices, which are renumbered on every load** — any finding must be keyed on node id or name.

## Code map

Server model (`common/data/src/main/java/org/thingsboard/server/common/data/`):
- `rule/RuleChainMetaData.java` — the wire graph: `firstNodeIndex`, `nodes`, `connections`, `ruleChainConnections`, `notes`, `version`; `addConnectionInfo(from,to,type)`.
- `rule/NodeConnectionInfo.java` — edge: `fromIndex`, `toIndex`, `type` (label).
- `rule/RuleChainConnectionInfo.java` — legacy cross-chain edge: `fromIndex`, `targetRuleChainId`, `type`, `additionalInfo`.
- `rule/RuleNode.java`, `rule/RuleChain.java` (`firstRuleNodeId`, `root`, `type`), `rule/RuleChainOutputLabelsUsage.java` (`labels: Set<String>` consumed elsewhere).
- `msg/TbNodeConnectionType.java` — the well-known labels: `Success`, `Failure`, `ACK`, `True`, `False`, `Other`, `To Root Rule Chain`.
- `relation/RelationTypeGroup.java` — `RULE_CHAIN`, `RULE_NODE`, `COMMON`, `EDGE_AUTO_ASSIGN_RULE_CHAIN`.

Node contract: `rule-engine/rule-engine-api/.../api/RuleNode.java` — `relationTypes()` (default `{Success, Failure}`), `inEnabled()`, `outEnabled()`, `customRelations()`, `ruleChainNode()`, `clusteringMode()`, `ruleChainTypes()`.

Cross-chain nodes: `rule-engine/rule-engine-components/.../flow/TbRuleChainInputNode.java` (target in config, `customRelations = true`, self-target rejected at runtime), `.../flow/TbRuleChainOutputNode.java` (`outEnabled = false`, name is the label).

Persistence (`dao/src/main/java/org/thingsboard/server/dao/`):
- `rule/BaseRuleChainService.java` — `saveRuleChainMetaData` (`:199`, version check then covering lock), `doSaveRuleChainMetaData` (`:235`, index→id, full relation replace), `loadRuleChainMetaData` (`:387`, id→index), `getRuleChainInputRelation` (`:370`), `getRuleNodeRelations` (`:503`), `getRuleChainToNodeRelations` (`:1007`).
- `service/validator/RuleChainDataValidator.java` — `validateMetaDataFieldsAndConnections` (`:85`), `validateCircles` (`:108`, `:121`), `validateRuleNode` (`:92`).
- `src/main/resources/sql/schema-entities.sql` — `rule_chain` (`:217`, incl. `first_rule_node_id`), `rule_node` (`:234`, no FK to `rule_chain`), `relation` (`:521`, where the edges actually live). Not in any code graph; read directly.

API and runtime (`application/src/main/java/org/thingsboard/server/`):
- `controller/RuleChainController.java` — metadata save/load, `getRuleChainOutputLabels` (`:170`), `getRuleChainOutputLabelsUsage` (`:184`).
- `service/rule/DefaultTbRuleChainService.java` — `getRuleChainOutputLabels` (`:64`), `getOutputLabelUsage` (`:76`), label-rename propagation (`:383`).
- `actors/ruleChain/RuleChainActorMessageProcessor.java` — `initRoutes` (`:159`), `onTellNext` (`:199`), `onRuleChainOutputMsg` (`:240`), `onRuleChainToRuleChainMsg` (`:259`), `pushToTarget` (`:342`), `contains` (`:376`).
- `actors/ruleChain/DefaultTbContext.java` — `input` (`:203`, stack push + loop guard), `output` (`:230`, stack pop), `enqueue` (`:241`), `tellSelf` (`:196`).

UI (`ui-ngx/src/app/`):
- `shared/models/rule-node.models.ts` — `RuleNodeDefinition` (`inEnabled`, `outEnabled`, `relationTypes`, `customRelations`, `ruleChainNode`), `LinkLabel`, `FcRuleNode`, `FcRuleEdge` (with multi-`labels`), `RuleNodeType` (incl. `INPUT`).
- `modules/home/pages/rulechain/rulechain-page.component.ts` — entry-point edge on load (`:653`) and on save (`:1698`), `validate()` (`:1647`), per-label edge split (`:1708`), `noInputNodes` in-degree logic (`:1047-1062`).
- `modules/home/pages/rulechain/rulechain-page.models.ts` — `FcRuleNodeModel {nodes, edges}`.
- `core/http/rule-chain.service.ts` — `getRuleNodeSupportedLinks` (`:143`), the only place labels are constrained.
- `core/services/import-export.service.ts` — `processOldRuleChainConnections` (`:740`), migrates `ruleChainConnections` to input nodes.

## Tests

- `dao/src/test/java/.../service/RuleChainServiceTest.java` — the core graph test: `testSaveRuleChainMetaData` (`:292`), `testUpdateRuleChainMetaData` (`:342`), `testUpdateRuleChainMetaDataWithCirclingRelation` (`:390`) and `...Relation2` (`:397`), `testSaveRuleChainWithInputNode` (`:432`), `testRuleChainGraphMutationsRunUnderCoveringLock` (`:322`).
- `dao/src/test/java/.../service/validator/RuleChainDataValidatorTest.java` — only `testValidateNameInvocation`. **No unit test for `validateCircles` / `validateMetaDataFieldsAndConnections`** — this is where a reachability check's tests would naturally go.
- `application/src/test/java/.../controller/RuleChainControllerTest.java` — metadata save/load, versioning, notes.
- `application/src/test/java/.../rules/flow/AbstractRuleEngineFlowIntegrationTest.java` — `testRuleChainWithTwoRules` (`:118`), `testTwoRuleChainsWithTwoRules` (`:221`): the only tests exercising cross-chain connections at runtime.
- `dao/src/test/java/.../service/CitusRelationWriteSerializationTest.java:131-147` — concurrent metadata re-save under the covering lock.
- `dao/src/test/java/.../agent/StepLinkedListUtilsTest.java:151-165, 445-448` — the existing orphan-detection tests, worth reading as a model.

## How to build and test

From `/workdir/repositories/thingsboard` (JDK 25, Maven 3.9.x; Node/Yarn come from `frontend-maven-plugin`):

```bash
# compile the modules this area spans
mvn -pl common/data,dao,application -am -DskipTests -Dpkg.skip=true test-compile

# the graph tests
mvn test -pl dao -Dtest='RuleChainServiceTest,RuleChainDataValidatorTest'
mvn test -pl application -Dtest='RuleChainControllerTest'

# license headers, required before committing
mvn license:format
```

The `dao` and `application` tests start real databases via Testcontainers, so Docker must be available; see CONTRIBUTING.md for the `min-api-version` workaround. The UI, from `ui-ngx`: `yarn install`, then `yarn lint` / `yarn build:prod`.

## Constraints and pitfalls

- **Optimistic locking.** `saveRuleChainMetaData` throws `EntityVersionMismatchException` when the supplied `version` differs from stored (`BaseRuleChainService.java:204`), surfacing as HTTP 409, which the UI handles by silently reloading. Any new validation must run **after** that check — as `validateMetaDataFieldsAndConnections` does at `:207` — or it will report graph errors against a stale payload.
- **Covering-lock ordering is load-bearing.** The whole save runs inside `relationWriteLock.withCoveringLock(ruleChainId, ...)` with the `rule_chain` row write pulled *inside and after* the relation writes, for Citus deadlock-freedom (`:219-227`, plus the contract in `RelationWriteLock`). Do not move the `ruleChainDetailsDao.save` or add new locked endpoints. One residual race is documented and accepted at `:262-268` (the cross-chain `Uses` row vs target-chain deletion).
- **`@Transactional` proxy re-entry.** `importTenantRuleChains` is deliberately non-transactional and must call back through the `self` proxy (`:116-124`, `:663-669`) because `withCoveringLock` asserts an active transaction.
- **Save is a destructive full replace**, so any "reject an invalid graph" rule has to be a pre-check; there is no partial-rollback semantics beyond the transaction.
- **The index-based wire format is public API** (`POST /api/ruleChain/metadata`), used by `rest-client`, the black-box tests, `InstallScripts` and the solution templates. `firstNodeIndex`/`fromIndex`/`toIndex` cannot change shape, and `ruleChainConnections` cannot be dropped from the DTO without breaking old import files.
- **Indices are not identifiers.** They are reassigned on every load by `(createdTime, id)`. Report findings by node id or name.
- **Edge sync.** `RuleChainMetaData` ships as JSON over the edge protocol (`EdgeMsgConstructorUtils.constructRuleChainMetadataUpdatedMsg`); older edge builds must still parse it. Edge chains also bypass the actor system entirely — `initRoutes`/`start` run only for `RuleChainType.CORE` — so validation must not assume CORE semantics.
- **Export/import and version control** round-trip the same DTO with external-id remapping, and `RuleChainImportService.isUpdateNeeded` does a full `RuleChainMetaData.equals` against a freshly loaded one, so adding a DTO field can cause spurious updates.
- **Stored data is messy by design.** Dangling relations are self-healing (`getRuleChainNodes:481`, `getRuleNodeRelations:507-525` delete rows whose endpoint is gone), legacy `RULE_NODE->RULE_CHAIN` rows may exist, and `first_rule_node_id` may point anywhere. A traversal must be defensive about all three rather than throwing.
- `rule_chain` and `rule_chain_details` map to the **same table row** (`RuleChainDetails extends RuleChain`; `JpaRuleChainDetailsDao` detaches to dodge a stale-version conflict).
- **Validating the graph is currently cheap to get wrong on cost.** Because `validateCircles` is already exponential in path count, adding another traversal in the same place without a visited set would compound an existing performance problem.

## Not confirmed

- Whether any production or edge client still POSTs `ruleChainConnections`. No in-repo main-code writer exists and the UI migrates it away, but the DTO field and the `doSaveRuleChainMetaData` branch are both live, so an external script or an old edge build could still send it.
- Whether `RULE_NODE -> RULE_CHAIN` relation rows actually exist in any upgraded deployment. The runtime handles them and the loader warns about them, but no migration was found that creates or removes them, so it is unclear whether that runtime branch is dead in practice.
- Whether `validateCircles` overflows the stack **inside the running server** specifically. The algorithm does so in a standalone replica under JDK 25 (`/workdir/probe/Probe.java`), and the server path is the same code, but it was not reproduced through an actual save, and the outcome depends on `HashMap` key iteration order, so the exact failure mode in production (`StackOverflowError` vs the clean `DataValidationException`) is not guaranteed for every cyclic input.
- Where `node.error` is assigned in the UI. Only the reads were located (`rulechain-page.component.ts:1508`, `:1651`, `:1893`; `item-buffer.service.ts:357`), so it is unclear what range of conditions the existing `isInvalid` gate actually covers.
- Edge-chain (`RuleChainType.EDGE`) runtime semantics. The actor layer skips those chains entirely, so this repo does not show how or whether an entry point is enforced on the edge side.
- **No Graphify graph was built.** `graphify update` core-dumps in this environment — on the full repository and equally on a 44-file rule-chain-only subset, so it is not a scale problem. Nothing under `/workspace/knowledge/graph/` exists, and the code map above comes entirely from reading files directly.

Source: commit 97b78f4989902ad8448d1dca573063fc4a5d40d4
