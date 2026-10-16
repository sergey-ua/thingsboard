# Tasks

## 1. Minimal test runner for plain TypeScript

- [ ] 1.1 Add `jest`, `ts-jest` and `@types/jest` to `ui-ngx/package.json` devDependencies and verify `yarn install` in `ui-ngx` still succeeds, including the `prepare`/`patch-package` step
- [ ] 1.2 Add a Jest config scoped to the rule-chain analysis spec files (ts-jest, `node` environment, the root `tsconfig.json` path aliases only as far as the files under test need them) and a `test` script, and verify `yarn test` in `ui-ngx` runs and reports zero tests rather than failing to start
- [ ] 1.3 Add a tsconfig that owns `**/*.spec.ts` and a spec-scoped block in `ui-ngx/eslint.config.mjs` declaring Jest's globals, and verify `yarn lint` passes with a spec file present
- [ ] 1.4 Verify the existing build is untouched: `yarn build:prod` in `ui-ngx` succeeds and `angular.json` has no `test` target

## 2. The finding model and the pure reachability check

- [ ] 2.1 Add `rulechain-analysis.models.ts` with `RuleChainFinding { checkType; nodeName?; explanation }` and the check-type enum, with the project SPDX licence header, and verify it compiles under the spec tsconfig
- [ ] 2.2 Add `rulechain-analysis.ts` with a connector-id → node index built from `node.connectors` and a BFS forward from the input connector id, free of Angular, jQuery and `ngx-flowchart`, and verify it imports nothing from `@angular/*` or `ngx-flowchart`
- [ ] 2.3 Report every node other than the input that the walk did not reach, using the i18n key whose value is `Not reachable from chain input` and an explanation that does not claim the node can never execute, and verify with a spec case asserting the exact text
- [ ] 2.4 Produce one chain-level finding with no `nodeName` when no edge leaves the input connector, instead of one per node, and verify with a spec case over a 3-node graph with no first node that exactly one finding comes back
- [ ] 2.5 Sort findings by check type then node name, placing a node-less finding first within its check type, and deduplicate to at most one finding per node per check, and verify with spec cases for ordering and for a node matching twice
- [ ] 2.6 Add the three acceptance-criteria unit tests in `rulechain-analysis.spec.ts` — a reachable node is not flagged, an unreachable node is flagged, and an all-reachable graph returns an empty list — plus a case that a node with no inbound connector is flagged (Q1 A), and verify `yarn test` passes

## 3. i18n strings

- [ ] 3.1 Append the new keys to the `"rulechain"` section of `ui-ngx/src/assets/locale/locale.constant-en_US.json` — Analyze label, drawer title, check-type label, the unreachable finding text, its explanation, the no-first-node message and the dismiss tooltip — and verify the file is still valid JSON
- [ ] 3.2 Verify the two acceptance-tested values are verbatim, with no markup or placeholder: `Not reachable from chain input` and `No issues found by these checks.`
- [ ] 3.3 Verify `git status --short` shows no other locale file changed

## 4. Analyze control and component wiring

- [ ] 4.1 Add the findings array, the drawer-open flag, `onAnalyze()` and `dismissFindings()` to `RuleChainPageComponent`, passing `ruleChainModel.nodes`, `ruleChainModel.edges` and `inputConnectorId` to the pure check, and verify `yarn build:prod` compiles
- [ ] 4.2 Add the Analyze control inside `mat-drawer-content.tb-rulechain-graph-content`, outside every `@if (!readonly)` block, and verify in the running UI as a user without RULE_CHAIN WRITE that it is present and enabled (C1)
- [ ] 4.3 Give the Analyze button its own offset on the right-offset ladder in `rulechain-page.component.scss`, extending the `:has()` fallbacks so no gap or overlap appears when the version-control and add-note buttons are absent, and verify visually for both a read-only viewer and a full-permission user
- [ ] 4.4 Verify `onAnalyze()` leaves the graph, `isDirtyValue`, `isInvalid` and the save FAB untouched: `git diff` touches none of `validate()`, `isInvalid`, `node.error` or `saveRuleChain()`, and Save stays enabled in the UI while findings are shown

## 5. Findings drawer

- [ ] 5.1 Add the bottom findings drawer markup as a sibling inside `tb-rulechain-graph-content`, not as a child of `mat-drawer-container`, and verify in the UI that the canvas keeps full width while it is open (C11)
- [ ] 5.2 Render one row per finding showing check type, node name and explanation with warning styling, and the node-less chain-level finding without a node name, and verify a row's appearance against the warning (not error) palette (C8)
- [ ] 5.3 Render exactly `No issues found by these checks.` and no rows when the list is empty, and verify by analyzing a fully connected chain (C10)
- [ ] 5.4 Make the drawer dismissible and verify in the UI that it opens and closes independently of the node-configuration panel, in both orders, and that it neither covers nor is covered by the footer save/revert FABs (C11)

## 6. Integration check

- [ ] 6.1 Run `yarn install`, `yarn lint`, `yarn test` and `yarn build:prod` in `ui-ngx` and verify all four succeed
- [ ] 6.2 In the running UI, analyze a chain after unsaved additions, edits and deletions and verify the findings reflect the canvas and not the last saved state (C2)
- [ ] 6.3 Verify `git status --short` lists only the files named in `design.md` — no server module, no other locale file, no `angular.json`, no `pom.xml`
