# Proposal

## Why

A rule chain can be saved with nodes that no message will ever reach: neither the Angular editor's
`validate()` (per-node `error` only) nor the server-side validators look at the shape of the graph, as
`rule-chain-graph` records in "There is no reachability or unreachable-node check anywhere". A user who
leaves a node disconnected gets no hint of it anywhere in the product. THIN-1 adds the first advisory,
client-side check and the UI it needs: an **Analyze** action on the rule-chain editor and a findings drawer
that reports nodes with no path from the chain input.

The check is **advice, not validation**. It never blocks Save and it never changes what the server accepts.

## What Changes

- An **Analyze** control on the rule-chain editor canvas, visible and enabled for every user who can open
  the editor — including a read-only viewer without RULE_CHAIN WRITE. It therefore sits deliberately
  outside the template's `@if (!readonly)` regions (C1).
- A dismissible **findings drawer at the bottom of the canvas**, one row per finding. The canvas keeps full
  width, and the drawer opens and closes independently of the `position="end"` node-configuration drawer
  (C11).
- **Check A, reachability**: a forward walk of the connections from the synthetic chain-input node over the
  in-memory `ruleChainModel`, so unsaved additions, edits and deletions are included with no extra work
  (C2). Every node not reached is reported with the exact wording **"Not reachable from chain input"**, and
  the explanation does not claim the node can never execute (C3).
- **Finding rows** show check type, node name and a short explanation, styled as a warning and not an error
  (C8). With no findings the drawer shows exactly **"No issues found by these checks."** (C10).
- Findings are **ordered by check type then node name**, and a node appears **at most once per check**.
- When the chain has **no first node at all** (no edge leaves the input), the drawer shows one chain-level
  message instead of one row per node — decided by the requester (Q2 B). The row format grows an optional
  node-less form to carry it.
- The reachability walk is extracted as a **pure, Angular-free function** and gets a **minimal Jest setup**
  with a `test` script, scoped to plain TypeScript only (decided: Q3 A). No Angular test tooling, no change
  to the Angular or Maven build.
- New English strings go into the existing `locale.constant-en_US.json` only; the other 27 locale files are
  untouched.

Not in this change (ticket's own exclusions): generator-node exclusions — a node that accepts no input is
flagged like any other unreached node (decided: Q1 A) — extra entry points, the Failure check,
click-to-navigate, and outdated-state handling.

No breaking changes: nothing existing changes behaviour, and no API or persisted shape is touched.

## Capabilities

### New Capabilities

None. The behaviour belongs to the existing rule-chain editor page, so it joins that capability.

### Modified Capabilities

- `rule-chain-editor-ui`: four requirements are **added** — the Analyze action available to read-only
  viewers, the dismissible bottom findings drawer, the reachability check over the live canvas graph, and
  the ordering/deduplication rule for findings. Two existing requirements are **modified**: "ui-ngx has no
  test runner" (a `test` script and a minimal Jest setup now exist for plain TypeScript) and "English UI
  strings live in one locale constant file" (the new keys and the two verbatim acceptance-tested strings).

Explicitly **not** modified: "Editor validation is per-node only and it gates Save" (this change touches
neither `validate()`, `isInvalid`, `node.error` nor `saveRuleChain()`, and a warning never disables Save),
"The editor opens read-only without RULE_CHAIN WRITE permission and then hides most editing controls" (the
new controls sit outside its guards, which the spec already records as established practice), and the whole
of `rule-chain-graph`, which is read-only background here — its "There is no reachability or unreachable-node
check anywhere" stays true, because this check is client-side and advisory.

## Impact

Angular UI module `ui-ngx` only; no server, API, database or dependency change.

- `src/app/modules/home/pages/rulechain/rulechain-page.component.{html,ts,scss}` — the Analyze control, the
  drawer markup and its styles, plus the findings state and handler on `RuleChainPageComponent`.
- new `src/app/modules/home/pages/rulechain/rulechain-analysis.models.ts` and `rulechain-analysis.ts` — the
  finding shape and the pure check, with `rulechain-analysis.spec.ts` beside them.
- `src/assets/locale/locale.constant-en_US.json` — new keys in the `"rulechain"` section.
- `package.json`, a new Jest config, `eslint.config.mjs` and a spec-owning tsconfig — the minimal runner.
- Read, not changed: `rulechain-page.models.ts`, `shared/models/rule-node.models.ts`,
  `shared/models/rule-chain.models.ts`.

## Traceability

Ticket: THIN-1
Criteria: C1, C2, C3, C8, C10, C11, plus the unit-test criterion (reachable node not flagged, unreachable node flagged, empty-result text)
