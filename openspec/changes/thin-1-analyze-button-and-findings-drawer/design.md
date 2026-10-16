# Design

## Context

See `proposal.md` — Why. The constraints that shape the approach are all recorded in the
`rule-chain-editor-ui` main spec and confirmed by reading the files at this branch's HEAD:

- `ruleChainModel` (`rulechain-page.component.ts:180`) is the live canvas graph. Its object identity is never
  replaced, the arrays are mutated in place, and nothing reloads it between a build and a save — so reading
  it satisfies C2 with no extra work.
- Edges carry **connector** ids, not node ids (`ts:633`–`:648`). Existing code resolves an end through
  `ruleChainCanvas.modelService.nodes.getNodeByConnectorId(...)` (`ts:208` and 14 more sites); an equivalent
  index can be built from `node.connectors` with no flowchart service.
- The root is the synthetic input node, first in `nodes`, `readonly: true`, empty `name`, with exactly one
  right connector whose id is `inputConnectorId + ''` (`ts:582`, `:596`). Its outgoing edge is found by
  `edge.source === inputConnectorId + ''` (`ts:659`, `:1698`), and there is at most one.
- `inputNodeComponent` has **no** `configurationDescriptor`, so every existing traversal guards on
  `component.type !== RuleNodeType.INPUT` before touching it.
- `ruleChainModel.notes` is a separate array; notes have no `connectors` and no `component`.
- `readonly` (`ts:127`) wraps four template regions (`html:14`, `:183`, `:195`, `:286`). Two canvas controls
  are already ungated, so an ungated control inside the content pane is established practice.
- `ui-ngx` has no test runner: no `test` script, no Jest/Karma/Jasmine, zero `*.spec.ts` under `src`.

Three questions were put to the requester and answered: unreachable nodes that accept no input are flagged
like any other (**Q1 A**); a chain with no first node yields one chain-level message rather than one row per
node (**Q2 B**); and the test runner is minimal, plain-TypeScript only (**Q3 A**).

## Goals / Non-Goals

**Goals:**
- Keep the reachability walk a pure function over plain data, so it is testable with no Angular, jQuery or
  `ngx-flowchart` setup.
- Shape the finding list for several checks from the start, because the ordering and deduplication rules in
  the spec are per check type even though only check A exists here.
- Add the new UI inside the canvas content pane, outside every `readonly` guard.

**Non-Goals:**
- Angular component or template tests. The drawer and the button are verified by hand for this change.
- Touching `validate()`, `isInvalid`, `node.error` or `saveRuleChain()` in any way.
- Any change to the other 27 locale files, to `angular.json`, or to the Maven build.

## Decisions

**A pure function over `{nodes, edges}`, not a service or a flowchart-aware helper.**
New `rulechain-analysis.ts` exports a single entry point taking the node and edge arrays plus the input
connector id, and returning the sorted, deduplicated finding list; `rulechain-analysis.models.ts` holds
`RuleChainFinding { checkType; nodeName?; explanation }` and the check-type enum. The function builds its own
connector-id → node index from `node.connectors`, does a BFS from the input connector, and reports every
node in `nodes` other than the input that the walk did not reach. Alternative considered: an Angular
`@Injectable` service calling `modelService.nodes.getNodeByConnectorId(...)`. Rejected — it would drag the
flowchart service and Angular's DI into the unit tests, which is exactly the cost Q3 A was chosen to avoid.
`notes` is never passed in, so excluding notes needs no code.

**`nodeName` is optional, to carry the chain-level finding (Q2 B).**
The no-first-node case produces one finding with no `nodeName` and its own explanation key. Ordering places
a node-less finding before the named ones of the same check type. Alternative considered: a required
`nodeName` with a sentinel value — rejected, it would print a fake node name in a row.

**Findings live on the component; the drawer is markup inside the content pane.**
`RuleChainPageComponent` gains a findings array, a drawer-open flag and an `onAnalyze()` that passes
`ruleChainModel.nodes`, `ruleChainModel.edges` and `inputConnectorId` straight to the pure function. The
Analyze control and the drawer go inside `<mat-drawer-content class="tb-rulechain-graph-content">`
(`html:182`), as siblings of the existing canvas controls and **outside** the `@if (!readonly)` blocks.
Alternative considered: a fourth child of `mat-drawer-container`. Rejected — Angular Material would lay it
out as another *horizontal* drawer, which breaks C11's full-width canvas; the main spec records this.

**Warning styling without `node.error`.**
Findings are rendered only in the drawer, with warning colours. Reusing `node.error` to surface a finding
would make `isInvalid` true and disable the save FAB, which contradicts "warnings never block Save".

**Minimal Jest, plain TypeScript only (Q3 A).**
A `test` script plus `jest` + `ts-jest` and a small config limited to the analysis spec file; a tsconfig that
owns `*.spec.ts` so ESLint's `parserOptions.project: true` resolves it, and a spec-scoped ESLint block
declaring Jest's globals. Alternative considered: full `jest-preset-angular`. Rejected for this change —
eight `patch-package` patches and a git-tag `ngx-flowchart` dependency make Angular test setup a large,
risky job for three tests. The spec records that an Angular component test still needs more work.

**The Analyze control takes its own slot on the existing right-offset ladder.**
`.tb-fullscreen-button` is at `right: 10px`, `.version-control-button` at `60px`, `.add-note-button` at
`110px` with a `:has()` fallback to `60px`. The Analyze button needs its own offset, and because it is
ungated while the other two are conditional, the fallback rules in the ladder need extending so no gap or
overlap appears for a read-only viewer.

## Files and symbols

| File | What changes |
|---|---|
| `ui-ngx/src/app/modules/home/pages/rulechain/rulechain-page.component.html` | Analyze control + findings drawer markup inside `mat-drawer-content.tb-rulechain-graph-content` (`:182`), outside the `@if (!readonly)` blocks at `:183`, `:194`/`:195`, `:286` |
| `…/rulechain-page.component.ts` | On `RuleChainPageComponent` (`:104`): findings state, drawer-open flag, `onAnalyze()`, `dismissFindings()`. Reads `ruleChainModel` (`:180`) and `inputConnectorId` (`:259`). Does not touch `validate()` (`:1647`), `isInvalid` (`:142`), `node.error`, `saveRuleChain()` (`:1659`) or `readonly` (`:127`) |
| `…/rulechain-page.component.scss` | Analyze button offset on the ladder (`:26`, `:37`, `:46`, `:55`) and the bottom drawer strip, which shares the corner with global `section.tb-footer-buttons` (`src/styles.scss:142`) |
| **new** `…/rulechain/rulechain-analysis.models.ts` | `RuleChainFinding`, check-type enum |
| **new** `…/rulechain/rulechain-analysis.ts` | The pure check: connector index, BFS, sort, dedupe |
| **new** `…/rulechain/rulechain-analysis.spec.ts` | The three required unit tests |
| `ui-ngx/src/assets/locale/locale.constant-en_US.json` | New keys appended to `"rulechain"` (`:7566`–`:7639`), including the two verbatim strings |
| `ui-ngx/package.json`, new Jest config, spec tsconfig, `ui-ngx/eslint.config.mjs` | The minimal runner |
| Read, not changed | `rulechain-page.models.ts` (`FcRuleNodeModel`, `:11`), `shared/models/rule-node.models.ts` (`FcRuleNode` `:345`, `FcRuleEdge` `:356`), `shared/models/rule-chain.models.ts` (`inputNodeComponent` `:78`) |

## How the work is tested

- **Unit (Jest, the three required cases):** a node reachable from the input is not flagged; a node with no
  path is flagged with the exact text `Not reachable from chain input`; a graph with no findings yields an
  empty list, which the drawer renders as `No issues found by these checks.` Plus the decided edge cases:
  no first node → one chain-level finding; a node with no inbound connector is flagged; sorting by check
  type then node name; one node reported once per check.
- **Lint:** `yarn lint` in `ui-ngx`, which must still pass with the spec file present.
- **By hand in the running UI:** C1 as a user without RULE_CHAIN WRITE (Analyze visible and enabled);
  C2 by adding and deleting nodes without saving; C8 and C10 in the drawer; C11 with the
  node-configuration panel open and closed, checking the canvas width.
- **Unchanged behaviour:** Save stays enabled with findings present; `yarn build:prod` still succeeds.

## Risks / Trade-offs

- **Adding a Jest setup to a module whose install depends on eight `patch-package` patches could break
  `yarn install` or `yarn lint`.** → Keep the dependency set to `jest` + `ts-jest` + `@types/jest`, change
  no existing script, and run `yarn install`, `yarn lint` and `yarn build:prod` before the change is called
  done.
- **The button and the drawer are not unit-tested (Q3 A).** → Their acceptance criteria are checked by hand
  against the running UI, and the spec records that Angular test tooling is still absent.
- **Flagging nodes that accept no input (Q1 A) shows a warning on most real chains that use a generator
  node.** → Accepted by the requester; the exclusion is a follow-up ticket.
- **The bottom drawer shares the canvas corner with the fixed `section.tb-footer-buttons` overlay
  (`z-index: 30`) and the details drawer (`z-index: 59`).** → Verify the drawer does not cover the save and
  revert FABs for a user who has them, and that it is not covered by the details drawer.
- **Connector ids are renumbered on every model rebuild.** → The findings are a snapshot of one run; a
  rebuild (reload, revert, save) makes them stale. Outdated-state handling is explicitly out of scope, so
  the drawer is simply dismissible.
