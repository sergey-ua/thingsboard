# rule-chain-editor-ui Specification

## Purpose
The Angular rule-chain editor page (`ui-ngx`, `RuleChainPageComponent`) is where a tenant administrator views and edits one rule chain as a flowchart. This spec records how that page behaves today: how it decides it is read-only and which controls it then hides, how its canvas and drawers are laid out, how the in-memory graph `ruleChainModel` represents nodes, notes and edges (edges reference *connector* ids, not node ids), what the editor validates and what that validation gates, and the fact that `ui-ngx` has no test runner at all. It is the behaviour an advisory, client-side graph check — for example an "unreachable node" warning shown on the canvas — has to fit into. The server side of the same feature is in `rule-chain-graph`.

## Requirements

### Requirement: The editor opens read-only without RULE_CHAIN WRITE permission and then hides most editing controls
The system SHALL compute `readonly` once, at component construction, as `!userPermissionsService.hasGenericPermission(Resource.RULE_CHAIN, Operation.WRITE)` (`rulechain-page.component.ts:127`), and SHALL still render the page and its canvas when it is true, hiding only the editing controls.

Four template regions are each wrapped in `@if (!readonly)`: the node-library `mat-drawer` (`rulechain-page.component.html:14`), the library-toggle mini-fab (`:183`), the add-note mini-fab (`:195`, itself nested in an import+permission guard at `:194`) and the whole `section.tb-footer-buttons` block holding the delete-selected, reset-debug-settings, save and revert FABs (`:286`, section at `:287`). Two canvas controls are **not** gated on `readonly`: the fullscreen `mat-icon-button` (`:217`) and the version-control mini-fab (`:209`), which is gated on a VERSION_CONTROL/RULE_CHAIN WRITE permission pipe instead. So an ungated control inside the canvas content is already established practice.

In TypeScript, `readonly` additionally marks every rule node read-only (`node.readonly = this.readonly`, `ts:650`; the synthetic input node is unconditionally `readonly: true`, `ts:590`), forces the node and link details panels read-only (`html:93`, `:109`, `:123`, `:145`, `:157`), suppresses the canvas context menu (`openRuleChainContextMenu`, `ts:705`), skips the library-search subscription (`ts:307`) and the library panel open/close (`ts:559`), and makes `onModelChanged()` a no-op so no edit marks the page dirty or re-validates (`ts:1160`). In `ngAfterViewInit` (`ts:322`–`331`) it also forces `modelService.isEditable = () => false`, replaces the edge click handler with `openLinkDetails`, and sets `draggable="false"` on every `.fc-connector` through jQuery.

#### Scenario: Viewer without write permission opens a rule chain
- **WHEN** a user whose role lacks RULE_CHAIN WRITE opens the rule-chain editor
- **THEN** `readonly` is `true`, the node library, library toggle, add-note button and footer FAB section are absent from the DOM, and the canvas, the fullscreen button and the flowchart itself are rendered

#### Scenario: Viewer drags a node on the canvas
- **WHEN** `readonly` is `true` and a canvas change fires `modelChanged`
- **THEN** `onModelChanged()` returns without setting `isDirtyValue` and without calling `validate()`

### Requirement: The canvas is the content pane of a horizontal three-part drawer container
The system SHALL lay the editor out as a single `<mat-drawer-container class="size-full">` (`html:13`) containing, in order, the node-library drawer (`mat-drawer.tb-rulechain-library`, `mode="side"`, `position="start"`, `disableClose`, initially opened — `html:15`–`:79`), the node-configuration drawer (`<mat-drawer class="tb-details-drawer" position="end" mode="over">`, `html:81`–`:181`) and `<mat-drawer-content class="tb-rulechain-graph-content">` (`html:182`–`:319`).

The details drawer's `[opened]` is bound to `isEditingRuleNode || isEditingRuleNodeLink || isEditingNote` (`html:82`) and `(closed)` calls `onDetailsDrawerClosed()`; because it is `mode="over"` it floats above the content instead of resizing it. Inside `mat-drawer-content` are the ungated/gated canvas control buttons, the context-menu trigger and `mat-menu`, `div.tb-absolute-fill.tb-rulechain-graph` wrapping `<fc-canvas #ruleChainCanvas>` (`html:224`–`:285`), and the footer FAB section. The SCSS rule `.mat-drawer-content.tb-rulechain-graph-content { overflow: hidden; z-index: 0; }` with `.tb-rulechain-graph { z-index: 0; overflow: auto; }` is at `rulechain-page.component.scss:176`–`:183`; the canvas fill comes from `tb-absolute-fill`, so it is positioned absolutely within the content pane rather than flowing in it.

#### Scenario: A rule node is opened for editing
- **WHEN** the user double-clicks a node and `isEditingRuleNode` becomes `true`
- **THEN** the `position="end"` `mode="over"` details drawer opens over the canvas and the canvas keeps its width

#### Scenario: A canvas control is added next to the existing ones
- **WHEN** another floating control is placed inside `tb-rulechain-graph-content`
- **THEN** it must take its own absolute offset: the existing controls are hand-placed on a right-offset ladder in `rulechain-page.component.scss` — `.tb-fullscreen-button` at `right: 10px` (`:26`), `.version-control-button` at `right: 60px` (`:37`), `.add-note-button` at `right: 110px` (`:46`) with a `:has()` fallback to `right: 60px` when the version-control fab is absent (`:55`) — all `top: 10px; z-index: 2`, while `.tb-library-node-btn` is at `top: 50%; z-index: 1000` (`:59`)

#### Scenario: Something is rendered at the bottom of the canvas
- **WHEN** content is placed at the bottom of `tb-rulechain-graph-content`
- **THEN** it shares that corner with the global overlay `section.tb-footer-buttons { position: fixed; right: 20px; bottom: 20px; z-index: 30; pointer-events: none; }` (`ui-ngx/src/styles.scss:142`), and with `mat-drawer.tb-details-drawer` which `styles.scss:1050` sizes to `z-index: 59` and 100%/80%/65% width by breakpoint

#### Scenario: A fourth sibling drawer would be laid out horizontally
- **WHEN** an element is added as a direct child of `mat-drawer-container`
- **THEN** Angular Material treats it as another horizontal drawer of that container, which is why canvas-local UI is added inside `mat-drawer-content.tb-rulechain-graph-content` instead

### Requirement: ruleChainModel is the live, unsaved graph of the canvas
The system SHALL hold the edited graph in `ruleChainModel: FcRuleNodeModel` (`ts:180`, initialised `{nodes: [], edges: [], notes: []}`) and SHALL bind it directly to `<fc-canvas [model]="ruleChainModel">` (`html:274`), so that every canvas mutation — adding a node by drag from the library, drawing or deleting an edge, deleting a selection, pasting, adding a note, creating a rule chain from a selection — is reflected in that object before any save.

`FcRuleNodeModel extends FcModel { nodes: Array<FcRuleNode>; edges: Array<FcRuleEdge> }` (`rulechain-page.models.ts:11`). `FcRuleNode extends FcRuleNodeType extends FcNode` and adds `ruleNodeId?`, `configuration?`, `error?`, `highlighted?`, `ruleChainType?` (`shared/models/rule-node.models.ts:337`, `:346`); `FcRuleEdge extends FcEdge { labels?: string[] }` (`:356`); `FcRuleNote extends FcNote` (`:364`). `FcNode`/`FcEdge`/`FcModel` come from the `ngx-flowchart` dependency and are not vendored in this repository.

`createRuleChainModel()` (`ts:574`) rebuilds `nodes`, `edges` and `notes` from the loaded `ruleChainMetaData` — called from `init()` (`ts:378`), `reloadRuleChain()` (`ts:1789`), `revertRuleChain()` (`ts:1795`) and after a successful non-import save (`ts:1744`); `saveRuleChain()` (`ts:1659`) projects the same object back to `RuleChainMetaData`, and `ruleChainMetaData` is only ever the last-persisted snapshot. The identity of the `ruleChainModel` object itself is never replaced — the arrays are reassigned or emptied in place — which is what keeps the `fc-canvas [model]` binding alive. Nothing between a build and a save reloads it, so a graph read from `ruleChainModel` includes unsaved work and excludes nodes the user has deleted.

Some actions mutate the arrays directly (`addRuleNode` `ts:1803`, paste `ts:1280`/`:1314`/`:1325`, `createNestedRuleChain` `ts:1139`, `addNote` `ts:1371`, `saveRuleNode` `ts:1511`, `saveRuleNodeLink` `ts:1520`, `saveNote` `ts:1435`) and some go through `ruleChainCanvas.modelService` (every deletion and all selection handling, plus the flowchart's own drag and edge-create). `modelService.model` *is* `ruleChainModel`, so both paths hit the same arrays; what differs is whether `modelChanged` fires, and the direct mutators therefore call `onModelChanged()` themselves.

#### Scenario: Reading the graph after unsaved edits
- **WHEN** the user adds a node and an edge and does not save
- **THEN** `ruleChainModel.nodes` and `ruleChainModel.edges` already contain them, and `isDirtyValue` is `true`

#### Scenario: The chain is reloaded
- **WHEN** `createRuleChainModel()` runs
- **THEN** `nodes`, `edges` and `notes` are replaced from metadata, `isDirtyValue` is reset to `false`, and `validate()` is called

### Requirement: Edges reference connector ids, never node ids
The system SHALL give each node a `connectors` array of `{type, id}` and SHALL store each edge as `{source, destination}` holding **connector** ids, so resolving an edge end to a node requires a connector lookup.

Connector ids are decimal strings from a single counter `nextConnectorID`, reset to 1 with `nextNodeID` at the start of `createRuleChainModel()` (`ts:575`–`576`) and incremented for every connector created there (`ts:637`, `:645`) and later when nodes are dropped or pasted (`ts:1112`, `ts:1275`). A metadata node gets a left connector only when its `nodeDefinition.inEnabled` (`ts:633`) and a right connector only when `outEnabled` (`ts:641`), so a node may have one, both or neither. Edge creation is constrained to right→left by `editCallbacks.isValidEdge` (`ts:205`).

Existing code resolves an edge end through the flowchart service: `ruleChainCanvas.modelService.nodes.getNodeByConnectorId(edge.source)` (`ts:208`, and 14 more times — `ts:920`, `:953`, `:954`, `:1022`, `:1030`, `:1031`, `:1065`, `:1066`, `:1191`, `:1222`, `:1223`, `:1700`, `:1704`, `:1705`). An equivalent lookup can be built from `ruleChainModel.nodes[].connectors` without that service.

#### Scenario: Resolving which nodes an edge joins
- **WHEN** code holds an `FcRuleEdge`
- **THEN** `edge.source` and `edge.destination` match entries in some node's `connectors[].id`, not any `node.id`

#### Scenario: A node that cannot receive input
- **WHEN** a node's component definition has `inEnabled: false`
- **THEN** it has no left connector and no edge can target it

#### Scenario: Several relation types join the same pair of nodes
- **WHEN** the loaded metadata has more than one connection between the same source and destination node
- **THEN** they are collapsed into **one** edge keyed `sourceId + '_' + destId`, whose `label` is the types joined by `' / '` and whose `labels` array holds them all (`ts:666`–`694`) — so one edge may stand for several relations

#### Scenario: The model is rebuilt
- **WHEN** `createRuleChainModel()` runs again (reload, revert, or after a save)
- **THEN** both counters restart at 1 and every connector is renumbered, so a connector id is unique within one build but not stable across builds and is never persisted

#### Scenario: Nodes are pasted
- **WHEN** `pasteRuleChainObjects` runs (`ts:1241`, buffer read at `ts:1256`)
- **THEN** the buffered nodes arrive with `connectors: []` (`core/services/item-buffer.service.ts:339``core/services/item-buffer.service.ts:339`, re-cleared `:439`) and get fresh ids from the never-reset `nextConnectorID` (`ts:1264`–`1278`), so pasted ids collide with nothing; a pasted input-sourced connection is re-pointed at `inputConnectorId` and any existing input edge is deleted (`ts:1288`–`1295`)

#### Scenario: A rule chain is created from a selection
- **WHEN** `createNestedRuleChain()` replaces the selected nodes with one rule-chain node (`ts:967`)
- **THEN** it allocates `ruleChainInputId`/`ruleChainOutputId` (`ts:1112`–`1113`) and rewrites surviving edges in place — `inputEdge.destination = ruleChainInputId` (`ts:1141`) and `outputEdge.source = ruleChainOutputId` with `labels` reduced to `[outputEdge.label]` (`ts:1144`–`1145`) — the one place an existing edge's endpoints change without a model rebuild

### Requirement: The chain input is a synthetic node with one outgoing connector
The system SHALL push a synthetic input node as the first entry of `ruleChainModel.nodes` in `createRuleChainModel()` (`ts:583`–`:601`), before the metadata nodes. It uses `inputNodeComponent` (component type `RuleNodeType.INPUT`), has an empty `name`, `readonly: true`, no `ruleNodeId`, fixed position `x: 50, y: 150`, and exactly one connector — a right connector whose id is `inputConnectorId + ''`, where `inputConnectorId = nextConnectorID++` (`ts:582`), so it is `'1'` for a freshly built model.

The edge leaving the input is identified by `edge.source === this.inputConnectorId + ''` — at `ts:659` when the model is built from `firstNodeIndex`, at `ts:210` and `ts:1291` when a new input edge replaces the old one (only one is allowed), and as `firstNodeEdge` at `ts:1698` when the metadata is saved. `saveRuleChain()` skips the input node when projecting nodes (`ts:1678`) and skips edges whose source node is the input (`ts:1706`), turning that edge into `firstNodeIndex` instead.

`inputNodeComponent` (`shared/models/rule-chain.models.ts:78`) has `type`, `configurationVersion`, `clusteringMode`, `name: 'Input'` and `clazz: 'tb.internal.Input'` but **no `configurationDescriptor`**, so reading `node.component.configurationDescriptor.nodeDefinition` on the input node throws. Every traversal in the component therefore tests `node.component.type !== RuleNodeType.INPUT` first (`ts:921`, `:1032`, `:1176`, `:1192`, `:1630`, `:1678`, `:1706`).

#### Scenario: Code inspects the node definition of every node
- **WHEN** a walk reads `component.configurationDescriptor` without excluding the input node
- **THEN** it throws, because the input node's component descriptor has no `configurationDescriptor`

#### Scenario: The chain has a first node
- **WHEN** the loaded metadata has `firstNodeIndex > -1` and that node has a left connector
- **THEN** one edge `{source: inputConnectorId + '', destination: <that left connector>}` exists in `ruleChainModel.edges`

#### Scenario: A second edge is drawn from the input
- **WHEN** the user drags a new edge out of the input node while one already exists
- **THEN** `createEdge` deletes the existing input edge first, so the input always has at most one outgoing edge

### Requirement: Notes are canvas annotations, not graph nodes
The system SHALL keep sticky notes in `ruleChainModel.notes` (`ts:183`, cleared `ts:387`, filled from `ruleChainMetaData.notes` at `ts:696`, round-tripped on save), separate from `nodes`. An `FcRuleNote` carries `content`, colours, border and markdown flags plus coordinates (`rule-node.models.ts:364`) and has no `connectors` and no `component`, so no edge can reach it and it takes no part in the graph. Note ids are GUIDs (`guid()` at `ts:1366`, re-assigned on paste at `ts:1324`), not the counter-based `rule-chain-node-N` ids of nodes. `addNote()` sets `isDirty = true` (`ts:1372`) without calling `validate()`, unlike the other direct mutations of `ruleChainModel`. Several call sites treat `notes` as possibly undefined (`ts:1217`, `:1320`, `:1368`) even though the field is initialised to `[]`.

#### Scenario: A note is added to the canvas
- **WHEN** `addNote()` pushes a note (`ts:1371`)
- **THEN** it appears in `ruleChainModel.notes`, never in `ruleChainModel.nodes`, and has no connectors

### Requirement: Editor validation is per-node only and it gates Save
The system SHALL implement `validate()` (`ts:1647`) as a `setTimeout(…, 0)` that resets `isInvalid` to `false`, then iterates `ruleChainModel.nodes` setting `isInvalid = true` for any node with a truthy `error` and refreshing that node's error tooltip. There is no traversal of `edges` and no graph-shape check of any kind in the editor.

`isInvalid` (`ts:142`) is read in exactly two places: it disables the save FAB (`html:303`, `[disabled]="(isLoading$ | async) || isInvalid || (!isDirtyValue && !isImport)"`) and it blocks the apply action (`ts:847`). `validate()` is called from `createRuleChainModel()` (`ts:701`) and from `onModelChanged()` (`ts:1161`) — i.e. after every edit, when not read-only. `updateRuleNodesHighlight()` (`ts:1594`) only sets `node.highlighted` from the name search and is not a validation. `validate()` is also called from `pasteRuleChainObjects` (`ts:1330`).

`node.error` is close to dead in this build: the only place that ever sets it is `ItemBufferService` preserving an existing value during copy (`core/services/item-buffer.service.ts:356`–`357`), and the only place that clears it is `saveRuleNode()` (`ts:1508`–`1510`), so in practice `isInvalid` stays `false`. Besides `validate()`, `node.error` drives the tooltipster error bubble in `updateNodeErrorTooltip()` (`ts:1893`–`1925`) — reusing that field to surface an advisory finding would newly make `isInvalid` true and disable Save.

The other `validate`-named members are unrelated form validation: `RuleNodeConfigurationComponent.validate()` (`shared/models/rule-node.models.ts:136`, declared on the interface at `:69`), `RuleNodeDetailsComponent.validate()` (`rulechain/rule-node-details.component.ts:154`). `editCallbacks.isValidEdge` (`ts:205`) is a connector-type check during drag. The closest existing traversal is `canCreateNestedRuleChain()` (`ts:947`), which builds a destination set from the *selected* edges only.

#### Scenario: A node has a configuration error
- **WHEN** some node in `ruleChainModel.nodes` has a non-empty `error`
- **THEN** `isInvalid` becomes `true` and the save FAB is disabled

#### Scenario: A node is disconnected from the rest of the chain
- **WHEN** a node has no edges at all
- **THEN** `validate()` reports nothing, `isInvalid` stays `false` and the chain can be saved (consistent with `rule-chain-graph`: "There is no reachability or unreachable-node check anywhere")

### Requirement: ui-ngx has no test runner
The system SHALL today provide no way to run a unit test in `ui-ngx`. `ui-ngx/package.json` scripts are `ng`, `start`, `build`, `build:prod`, `build:types`, `build:icon-metadata`, `lint`, `prepare` — there is no `test` script; neither `dependencies` nor `devDependencies` contains Jest, Karma, Jasmine or `@angular/platform-browser-dynamic`-based test tooling beyond what the build needs; `angular.json` defines no `test` target; and `find ui-ngx/src -name '*.spec.ts'` returns nothing. `src/tsconfig.app.json` nevertheless already excludes `test.ts` and `**/*.spec.ts` from the application compilation, so spec files added under `src` do not enter the Angular build. Linting is ESLint 9 with `angular-eslint` through `yarn lint` (`eslint.config.mjs`). `ui-ngx/pom.xml` drives the frontend with `frontend-maven-plugin` (yarn 1.22.22, `pom.xml:61`): `yarn install` (`:65`–`70`) and, in the `yarn-build` profile, `run build:prod` (`:136`–`141`) — no test goal. Two things complicate adding a runner: the root `tsconfig.json` uses `module: "es2020"` with `moduleResolution: "node"` and a set of path aliases (`@app/*`, `@core/*`, `@shared/*`, `@home/*`, plus file-level aliases for `jquery`, `ace`, `tooltipster`, `jszip`, `jstree`) that any runner has to mirror; and installation itself depends on `prepare` running `patch-package --error-on-fail` over the eight patches in `ui-ngx/patches/`, with `ngx-flowchart` pulled straight from a git tag (`package.json:88`). A check written as a dependency-free function over `{nodes, edges}` therefore needs no Angular, TestBed, jQuery or flowchart setup to test.

#### Scenario: Running the UI tests
- **WHEN** `yarn test` is run in `ui-ngx`
- **THEN** it fails: no such script exists

#### Scenario: A spec file is added under src
- **WHEN** a `*.spec.ts` file is placed under `ui-ngx/src`
- **THEN** `tsconfig.app.json`'s `exclude` keeps it out of `ng build`, but it also belongs to no TypeScript project, which matters because `eslint.config.mjs` lints `**/*.ts` with `parserOptions.project: true`

#### Scenario: A spec file uses Jest globals
- **WHEN** `describe`/`it`/`expect` appear in a linted file
- **THEN** `eslintJS.configs.recommended`'s `no-undef` flags them unless a spec-scoped block declaring Jest globals is added to `eslint.config.mjs`

#### Scenario: The Maven build runs the UI
- **WHEN** `ui-ngx/pom.xml` drives the frontend through `frontend-maven-plugin`
- **THEN** it runs `yarn install` and `yarn build` only and invokes no test goal, so adding a `test` script does not change the Maven build

### Requirement: English UI strings live in one locale constant file
The system SHALL read UI strings through the `translate` pipe against `ui-ngx/src/assets/locale/locale.constant-en_US.json`, a single nested JSON object whose `"rulechain"` section starts at line 7566 and whose `"rulenode"` section starts at line 7640. Keys are dot paths into it, e.g. `{{'rulechain.add-note' | translate}}` (`html:200`), `{{'rulechain.note' | translate}}` (`html:167`), `{{'rulenode.search' | translate}}` (`html:37`). The directory holds 28 locale files (`ar_AE` … `zh_TW`); non-English locales are separate files and untranslated keys fall back. Within `"rulechain"` the keys are not sorted — they are grouped by feature, with the most recently added feature's keys appended last (the note keys run from `add-note` to `note-custom-css`, ending the section at line 7639).

The template uses three translation forms: the `translate` pipe inside attribute interpolation (`matTooltip="{{'rulechain.add-note' | translate}}"`, `html:200`); a ternary key then the pipe for toggles (`{{ (drawer?.opened ? 'rulenode.close-node-library' : 'rulenode.open-node-library') | translate }}`, `html:189`); and the `translate` structural attribute where the element's text *is* the key (`<span translate>{{menuItem.value}}</span>`, `html:264`) — which is how every context-menu label is translated, `RuleChainMenuItem.value` holding a raw key. In TypeScript, `translate.instant(...)` supplies hotkey descriptions (`ts:400`–`499`), context-menu subtitles (`ts:744`) and tooltip bodies (`ts:1526`–`1575`).

#### Scenario: A key is added to English only
- **WHEN** a new key is added under `"rulechain"` in `locale.constant-en_US.json` and nowhere else
- **THEN** English shows the new string and other locales fall back to it

## Code map
- `ui-ngx/src/app/modules/home/pages/rulechain/rulechain-page.component.ts`: the editor component — `readonly` gate, `ruleChainModel`, `createRuleChainModel()`, connector/`inputConnectorId` bookkeeping, `validate()`/`isInvalid`, `saveRuleChain()`, clipboard and context-menu actions.
- `ui-ngx/src/app/modules/home/pages/rulechain/rulechain-page.component.html`: the `mat-drawer-container` layout — library drawer, `position="end"` details drawer, `mat-drawer-content.tb-rulechain-graph-content` with the canvas controls, `fc-canvas` and the footer FAB section; the four `@if (!readonly)` regions.
- `ui-ngx/src/app/modules/home/pages/rulechain/rulechain-page.component.scss`: page styles; the content-pane rule at line 176 and flowchart node/note styling.
- `ui-ngx/src/app/modules/home/pages/rulechain/rulechain-page.models.ts`: `FcRuleNodeModel`, `FcRuleNodeTypeModel` and the context-menu interfaces.
- `ui-ngx/src/app/shared/models/rule-node.models.ts`: `FcRuleNode`, `FcRuleNodeType`, `FcRuleEdge`, `FcRuleNote`, `RuleNodeType`, rule-node type descriptors and the `RuleNodeConfigurationComponent` contract.
- `ui-ngx/src/app/shared/models/rule-chain.models.ts`: `RuleChainMetaData` (including `notes`), `inputNodeComponent` (line 78, no `configurationDescriptor`) and `unknownNodeComponent`.
- `ui-ngx/src/app/core/services/item-buffer.service.ts`: copy/paste buffer for rule-chain objects; strips `connectors`, stores `componentClazz`, preserves `node.error`.
- `ui-ngx/src/styles.scss`: global overlays that share the canvas corners — `section.tb-footer-buttons` (line 142) and `mat-drawer.tb-details-drawer` sizing (line 1050).
- `ui-ngx/eslint.config.mjs`: the only quality gate for `ui-ngx` (`yarn lint`); types every `**/*.ts` via `parserOptions.project: true`.
- `ui-ngx/pom.xml`: frontend-maven-plugin wiring — `yarn install` and `build:prod` only, no test goal.
- `ui-ngx/src/assets/locale/locale.constant-en_US.json`: English UI strings; `"rulechain"` at 7566, `"rulenode"` at 7640.
- `ui-ngx/package.json`: scripts and dependencies — no `test` script and no test framework.
- `ui-ngx/src/tsconfig.app.json`: application TypeScript config; excludes `test.ts` and `**/*.spec.ts`.

Source: commit 1cb3358408d3c8864477750f2ef93b0dbf3d2fc7
