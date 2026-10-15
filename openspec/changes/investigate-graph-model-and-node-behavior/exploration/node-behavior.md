# Node behavior

How a node type declares its outputs, how `Failure` is supported, and which nodes are triggered independently of any incoming edge.

What the ticket needs from this area: the declared-label pipeline and the set of independently-triggered node types, so a reachability warning does not contradict either. These are exploration notes, not a spec: no requirements, scenarios or deltas. No main spec exists for this domain yet. The graph-model side of the same change is in `graph-model.md`; this file does not repeat its cross-chain detail.

## Current behaviour

### A node type's outputs are declared by annotation, then rewritten on the server

A node type is a class annotated `@RuleNode` (`rule-engine/rule-engine-api/.../api/RuleNode.java`). The members that shape its edges are `relationTypes()` (default `{TbNodeConnectionType.SUCCESS, TbNodeConnectionType.FAILURE}`), `customRelations()` (default `false`), `inEnabled()` and `outEnabled()` (both default `true`), plus `ruleChainNode()`, `ruleChainTypes()` (default `{CORE, EDGE}`), `clusteringMode()`, `hasQueueName()`, `hasSecrets()` and `version()`. The annotation is the only declaration; there is no registry file and no interface method carrying this.

`AnnotationComponentDiscoveryService` (`application/.../service/component/`) is the whole pipeline from annotation to what any client sees:

1. `@PostConstruct init()` (`:73`) classpath-scans the packages in `plugins.scan_packages` for `@RuleNode` and fills `ruleNodeClasses: Map<String className, RuleNodeClassInfo>`. A class that fails to load is only `log.warn`-ed and silently absent. Under the `install` Spring profile `discoverComponents()` is skipped, so descriptors are not persisted during install.
2. `registerRuleNodeComponents()` (`:109`) calls `scanAndPersistComponent` per class, with up to `MAX_OPTIMISITC_RETRIES = 3` attempts and a 1 s sleep between them; a class that still fails throws and aborts startup.
3. `scanAndPersistComponent` (`:169`) builds a `ComponentDescriptor` (`common/data/.../plugin/ComponentDescriptor.java`): type, name, scope, clusteringMode, hasQueueName, hasSecrets, clazz, `configurationVersion` from the annotation's `version()`, and `configurationDescriptor = {"nodeDefinition": <NodeDefinition>}`.
4. `prepareNodeDefinition` (`:209`) fills `NodeDefinition` (`rule-engine/rule-engine-api/.../api/NodeDefinition.java`): details, description, `inEnabled`, `outEnabled`, `relationTypes`, `customRelations`, `ruleChainNode`, `defaultConfiguration` (a fresh `configClazz` instance's `defaultConfiguration()`, so **the config class must have a no-arg constructor** or startup fails), uiResources, configDirective, icon, iconUrl, docUrl.
5. **`getRelationTypesWithFailureRelation` (`:230-242`) is the one place the declared set is rewritten**, and it is not a pure copy:
   - For `TbOriginatorTypeSwitchNode` it appends `EntityType.NORMAL_NAMES` (every entity type's normal name).
   - For `TbMsgTypeSwitchNode` it appends `TbMsgType.NODE_CONNECTIONS` plus `TbNodeConnectionType.OTHER`.
   - Then, for **every** node type, if the list does not already contain `Failure`, it appends `Failure`.

   Both switch nodes declare `relationTypes = {}` with the comment "should always be empty. We add the relation types for this node in AnnotationComponentDiscoveryService" — the hard-coded `clazz` equality is load-bearing and a third such node would need another branch here. The consequence that matters for the ticket: **the annotation's `relationTypes` is never the authoritative label set, and `Failure` is a legal outgoing label on every node type without exception.**

Persistence is by `clazz`: `componentDescriptorService.findByClazz(SYS_TENANT_ID, clazzName)`; absent ⇒ save, equal ⇒ reuse the persisted row, different ⇒ `deleteByClazz` then save with the **old id reassigned** (`:195-206`). So an annotation change (including a changed label set) replaces the row in place and keeps its `ComponentDescriptorId`; rule nodes reference the descriptor by `type`/`configuration`, not by that id, so nothing dangles — but there is a window after the delete and before the save in which no descriptor exists for the class, and `ComponentDescriptor.equals` is what decides whether the rewrite happens at all.

Readers: in-memory `components`/`coreComponentsMap`/`edgeComponentsMap` via `getComponent(clazz)` and `getComponents(type, ruleChainType)` (`:251-296`), served over REST by `ComponentDescriptorController` (`GET /api/component/{clazz}`, `/api/components/{type}`, `/api/components`). `putComponentIntoMaps` (`:138`) splits CORE and EDGE by `ruleChainTypes()`, with a reflective `getMethod("ruleChainTypes")` fallback for extensions compiled before 3.3.

### Where the declared labels are actually enforced: only the UI

`ui-ngx/src/app/core/http/rule-chain.service.ts: getRuleNodeSupportedLinks` (`:143`) turns `component.configurationDescriptor.nodeDefinition.relationTypes` into the `LinkLabel` map offered when a link is drawn, and `ruleNodeAllowCustomLinks` (`:155`) returns `nodeDefinition.customRelations`. Both are consumed only at `rulechain-page.component.ts:222-223` (creating an edge) and `:1197-1198` (editing one). The TypeScript mirror of `NodeDefinition` is `RuleNodeDefinition` in `ui-ngx/src/app/shared/models/rule-node.models.ts` (`:41-44`).

Nothing on the server validates an edge's label against the source node's declared set — `RuleChainDataValidator` discards labels entirely (see `graph-model.md`). With `customRelations = true` (`TbDeviceTypeSwitchNode`, `TbAssetTypeSwitchNode`, `TbJsSwitchNode`, `TbRuleChainInputNode`, `TbDeviceProfileNode`) the declared set is a suggestion and arbitrary labels are legal. So the UI's offered set is the only existing constraint surface, and it is the surface any new warning has to agree with: a warning derived from `@RuleNode(relationTypes)` would under-approximate for those five, and one derived from the stored edges over-approximates (a switch whose script never returns a branch).

### `Failure` support

`Failure` is routed like any other label — `initRoutes`/`nodeRoutes` and the case-insensitive `contains` match (see `graph-model.md`) make no distinction. A node emits it either explicitly via `ctx.tellNext(msg, FAILURE)` or implicitly: `DefaultTbContext.tellFailure` (`:394-401`) sends `RuleNodeToRuleChainTellNextMsg` with `Collections.singleton(FAILURE)` and a failure message, and `RuleNodeActorMessageProcessor` wraps **every** `tbNode.onMsg` in `try/catch (Exception e) { ctx.tellFailure(...) }` (`:125-129`, `:148-152`). Any node type can therefore produce a `Failure` message at runtime, which is consistent with the discovery service appending the label to every type.

The one asymmetry is at the far end, in `RuleChainActorMessageProcessor.onTellNext` when `relationsCount == 0` (`:299-311`): if the relation types include `Failure` the message is `onFailure`-ed with a `RuleNodeException` (named rule node if its actor is still known, a generic `RuleEngineException` otherwise); otherwise it is `onSuccess`-ed and dropped. So an unrouted failure is reported and an unrouted success is not.

### Independently triggered nodes

Two separate mechanisms, and `inEnabled` is not a proxy for either:

- **`inEnabled = false`** — declared by exactly three types at this commit: `TbMsgGeneratorNode` (`.../debug/`, `:53`), `TbAlarmsCountNode` (`.../analytics/latest/alarm/`, `:40`) and `TbAggLatestTelemetryNode` (`.../analytics/latest/telemetry/`, `:38`); the latter two are the deprecated versions. These can never have an incoming edge — the UI only draws a left connector when `inEnabled` — so they are entry points by construction and must never be reported unreachable.
- **`ctx.tellSelf(msg, delayMs)`** — declared on `TbContext`, implemented in `DefaultTbContext` (`:197-200`) as `mainCtx.scheduleMsgWithDelay(selfActor, new RuleNodeToSelfMsg(...), delayMs)`, with a `//TODO: add persistence layer` (timers are in-memory and do not survive a restart). Callers in `rule-engine-components`: `TbMsgCountNode`, `TbMsgGeneratorNode`, `TbSimpleAggMsgNode`, `TbDeviceProfileNode`, `TbMsgDelayNode`, `TbMsgDeduplicationNode`, `TbAbstractLatestNode` (the base of the two `inEnabled = false` analytics nodes) and `TbAggLatestTelemetryNodeV2`.

Only `TbMsgGeneratorNode` and the `TbAbstractLatestNode` pair are in both sets. The other `tellSelf` users do have an input and use the timer to re-deliver or flush a message they already received (delay, deduplication, count, aggregate stream, device profile), so **`tellSelf` alone is not the test for "runs without an incoming edge"**: it tells you the node has its own clock, not its own source. The property a warning should key on is `inEnabled == false` — that is the only declared, statically readable statement that a node type takes no input, it is exactly the three types above, and it is already what the UI uses to decide whether to draw an input connector. `tellSelf` belongs in the reasoning about whether a node *does* anything (an `inEnabled = false` generator with no outgoing edge still fires, pointlessly), not about whether it is reachable.

`TbAggLatestTelemetryNodeV2` is worth noting as the direction of travel: it replaces an `inEnabled = false` node with an input-driven one (`inEnabled` default, triggered "based on incoming event"), so the three-type set is shrinking, not fixed, and should be computed rather than hard-coded.

### `TbRuleChainOutputNode`

The only `outEnabled = false` type (`.../flow/TbRuleChainOutputNode.java:22`). It takes the default `relationTypes`, which is moot: it never routes inside its own chain, and the UI draws no right connector. Its outgoing label lives in the **caller**, as the node's own name (`ctx.output(msg, ctx.getSelf().getName())`), consumed through `GET /api/ruleChain/{id}/output/labels`. For this area that means a node with no outgoing edge is not automatically useless, and `outEnabled = false` must suppress any "no outgoing connection" warning exactly as `inEnabled = false` must suppress the incoming one. The cross-chain routing itself is in `graph-model.md`.

## Code map

Declaration:
- `rule-engine/rule-engine-api/.../api/RuleNode.java` — the annotation; `relationTypes` default `{Success, Failure}`, `inEnabled`/`outEnabled`/`customRelations`/`ruleChainNode`/`ruleChainTypes`/`version`/`hasQueueName`/`hasSecrets`.
- `rule-engine/rule-engine-api/.../api/NodeDefinition.java` — the DTO carrying the declaration to clients; plain `@Data`, no validation.
- `application/.../service/component/RuleNodeClassInfo.java` — class + annotation pair, `isVersioned()` (it lives in `application`, not in `rule-engine-api`).

Pipeline (`application/src/main/java/org/thingsboard/server/service/component/`):
- `AnnotationComponentDiscoveryService.java` — `init` (`:73`), `discoverBeansByAnnotationType` (`:89`), `registerRuleNodeComponents` (`:109`), `putComponentIntoMaps` (`:138`, CORE/EDGE split), `scanAndPersistComponent` (`:169`, descriptor diff + `deleteByClazz` and re-save at `:195-206`), `prepareNodeDefinition` (`:209`), **`getRelationTypesWithFailureRelation` (`:230-242`)**, `getComponent`/`getComponents` (`:251-296`).
- `ComponentDiscoveryService.java` — the interface the rest of the server sees.

Node types that bend the rules (`rule-engine/rule-engine-components/src/main/java/org/thingsboard/rule/engine/`):
- `filter/TbOriginatorTypeSwitchNode.java:15`, `filter/TbMsgTypeSwitchNode.java:17` — `relationTypes = {}`, filled in by the discovery service.
- `filter/TbJsSwitchNode.java:25` — `relationTypes = {}` with `customRelations = true`; labels come from the script at runtime.
- `filter/TbDeviceTypeSwitchNode.java`, `filter/TbAssetTypeSwitchNode.java`, `flow/TbRuleChainInputNode.java`, `profile/TbDeviceProfileNode.java` (`:42-43`, a declared set *and* `customRelations = true`) — the other `customRelations = true` types.
- `debug/TbMsgGeneratorNode.java:53`, `analytics/latest/alarm/TbAlarmsCountNode.java:40`, `analytics/latest/telemetry/TbAggLatestTelemetryNode.java:38` — the three `inEnabled = false` types.
- `flow/TbRuleChainOutputNode.java:22` — the only `outEnabled = false` type.
- `analytics/latest/TbAbstractLatestNode.java`, `action/TbMsgCountNode.java`, `analytics/incoming/TbSimpleAggMsgNode.java`, `delay/TbMsgDelayNode.java`, `deduplication/TbMsgDeduplicationNode.java`, `profile/TbDeviceProfileNode.java`, `analytics/latest/telemetry/TbAggLatestTelemetryNodeV2.java` — the `tellSelf` callers.

Labels and runtime:
- `common/data/.../msg/TbNodeConnectionType.java` — `Success`, `Failure`, `ACK`, `True`, `False`, `Other`, `To Root Rule Chain`.
- `common/data/.../msg/TbMsgType.java:90` — `NODE_CONNECTIONS`. `common/data/.../EntityType.java:106-108` — `NORMAL_NAMES`.
- `application/.../actors/ruleChain/DefaultTbContext.java` — `tellNext` (`:189`), `tellSelf` (`:197`), `tellFailure` (`:394`).
- `application/.../actors/ruleChain/RuleNodeActorMessageProcessor.java:125-129`, `:148-152` — the `catch` that turns any exception into `Failure`; also the `maxRuleNodeExecutionsPerMessage` guard.
- `application/.../actors/ruleChain/RuleChainActorMessageProcessor.java:299-311` — the unrouted-`Failure` asymmetry.
- `application/.../actors/ActorSystemContext.java:951-954` — debug output keyed on the relation types, incl. the failures-only mode.

Persistence and API:
- `common/data/.../plugin/ComponentDescriptor.java` (`equals` decides whether the row is rewritten), `.../plugin/ComponentType.java`, `.../plugin/ComponentScope.java`, `.../plugin/ComponentClusteringMode.java`.
- `dao/src/main/java/org/thingsboard/server/dao/component/` — `BaseComponentDescriptorService`, `ComponentDescriptorDao`, `JpaComponentDescriptorDao`.
- `application/.../controller/ComponentDescriptorController.java` — `getComponentDescriptorByClazz` (`:46`), `getComponentDescriptorsByType` (`:59`), `getComponentDescriptorsByTypes` (`:73`).

UI:
- `ui-ngx/src/app/core/http/rule-chain.service.ts` — `getRuleNodeSupportedLinks` (`:143`), `ruleNodeAllowCustomLinks` (`:155`), `ruleNodeSourceRuleChainId` (`:159`), `getRuleNodeComponentByClazz` (`:131`, synthesises an `unknownNodeComponent` for an unrecognised class).
- `ui-ngx/src/app/shared/models/rule-node.models.ts:41-44` — `RuleNodeDefinition`.
- `ui-ngx/src/app/modules/home/pages/rulechain/rulechain-page.component.ts:222-223`, `:1197-1198` — the only consumers of the label set.

## Tests

- `dao/src/test/java/.../component/BaseComponentDescriptorServiceTest.java` — descriptor CRUD by clazz/type, the save-and-find path `scanAndPersistComponent` uses.
- `dao/src/test/java/.../sql/component/JpaBaseComponentDescriptorDaoTest.java`, `dao/src/test/java/.../service/validator/ComponentDescriptorDataValidatorTest.java` — dao and bean validation.
- `application/src/test/java/.../controller/ComponentDescriptorControllerTest.java` — the REST surface clients read the labels from.
- No test exists for `AnnotationComponentDiscoveryService` itself, and therefore **none for `getRelationTypesWithFailureRelation`** — nothing asserts that `Failure` is appended, nor the two hard-coded switch-node branches. There is no test directory `application/src/test/java/.../service/component/`. A change that relies on the appended-`Failure` rule should add the first one there.
- `application/src/test/java/.../rules/flow/AbstractRuleEngineFlowIntegrationTest.java` — runtime routing incl. failure paths.

## How to build and test

From `/workdir/repositories/thingsboard` (JDK 25, Maven 3.9.x; Node/Yarn via `frontend-maven-plugin`):

```bash
# the modules this area spans
mvn -pl common/data,rule-engine/rule-engine-api,rule-engine/rule-engine-components,dao,application -am \
    -DskipTests -Dpkg.skip=true test-compile

mvn test -pl dao -Dtest='BaseComponentDescriptorServiceTest,ComponentDescriptorDataValidatorTest'
mvn test -pl application -Dtest='ComponentDescriptorControllerTest'

mvn license:format    # required before committing
```

`dao` and `application` tests start databases via Testcontainers, so Docker must be available (see CONTRIBUTING.md for the `min-api-version` workaround). UI, from `ui-ngx`: `yarn install`, then `yarn lint` / `yarn build:prod`.

## Constraints and pitfalls

- **Never treat `@RuleNode(relationTypes)` as the label set.** Only `NodeDefinition.relationTypes` after `getRelationTypesWithFailureRelation` is what clients see, and the two switch nodes declare `{}` on purpose. Anything server-side that needs the real set must go through `ComponentDiscoveryService.getComponent(clazz)`, not reflection on the annotation.
- **`Failure` is always a legal label**, on every node type, and is produced for any uncaught exception. A check that treats a `Failure` edge as exceptional, or that flags `Failure` as "not declared by this node", will be wrong for every node type.
- **`customRelations = true` (five types) means arbitrary labels are legal**, so any label-based inference is an approximation there; say which way it errs.
- **Labels are strings, matched case-insensitively at runtime but case-sensitively in the `relation` primary key** (see `graph-model.md`). Comparisons against the declared set must use the same rule as the runtime, `equalsIgnoreCase`.
- **`inEnabled`/`outEnabled` are the suppression conditions**, not `tellSelf`. Compute the sets from the descriptors at runtime; hard-coding the three class names will rot (`TbAggLatestTelemetryNodeV2` already shows the direction).
- **Descriptors are system-tenant-scoped and discovered at startup.** Nothing can be asserted about them before `discoverComponents()` has run, and it is deliberately skipped under the `install` profile. A validator that resolves a node's declared labels must tolerate `getComponent(clazz)` returning empty — for a third-party extension class, a class that failed to load (`log.warn` only), or an installing server. The UI already does, by substituting `unknownNodeComponent`.
- **`ComponentDescriptor.equals` drives the rewrite path.** Adding a field to `ComponentDescriptor` or `NodeDefinition` makes every descriptor unequal on the next startup, so every row is deleted and re-saved once. Harmless, but it is a startup-time write storm worth knowing about.
- **Extension compatibility.** `putComponentIntoMaps` reflectively tolerates annotations compiled before 3.3 without `ruleChainTypes`; new members added to `@RuleNode` must have defaults and must not be read unconditionally for the same reason.
- `EDGE`-type chains get descriptors through `edgeComponentsMap` but never run through the actor system, so the `Failure` routing described above is CORE-only semantics.

## Not confirmed

- Whether any first-party or extension node type outside `rule-engine/` declares `inEnabled = false`. The grep covered `rule-engine/`; the scan is driven by the `plugins.scan_packages` property, so an out-of-tree extension could add one, and the set must therefore be computed at runtime rather than taken as exactly three.
- Whether the `deleteByClazz`-then-save window at `:195-206` is observable. It runs inside component discovery during startup and the surrounding transaction boundaries of `BaseComponentDescriptorService` were not traced, so it is unclear whether a concurrent REST read can see the class missing. The `MAX_OPTIMISITC_RETRIES` loop suggests concurrent discovery across microservices was a real problem.
- What a UI client does when a stored edge carries a label that is *not* in the node's current `relationTypes` (after a node type's annotation lost a label). `getRuleNodeSupportedLinks` only builds the offered map; whether the existing edge is rendered, flagged or silently kept was not traced, and this is the nearest existing analogue to the warning the ticket wants.
- Where `node.error` is assigned in the UI — only reads were found (see `graph-model.md`), so the existing per-node warning channel's actual trigger conditions are unknown.
- Whether `scheduleMsgWithDelay` timers are re-armed anywhere on restart. `tellSelf` carries a `//TODO: add persistence layer`, implying not, but no restart handling was located, so "an `inEnabled = false` generator is always live" holds only while its actor has been started.
- **No Graphify graph was built.** `graphify update` core-dumps in this environment — on the full repository and equally on a 44-file rule-chain-only subset, so it is not a scale problem. Nothing exists under `/workspace/knowledge/graph/`, and this code map comes entirely from reading files.

Source: commit 97b78f4989902ad8448d1dca573063fc4a5d40d4
