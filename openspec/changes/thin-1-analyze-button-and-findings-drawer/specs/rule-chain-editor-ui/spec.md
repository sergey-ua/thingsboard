# Spec Delta

## ADDED Requirements

### Requirement: An Analyze action on the editor canvas is available to every user who can open the editor
The system SHALL offer an **Analyze** control on the rule-chain editor canvas that is rendered and enabled
for every user who can open the editor, including a user without RULE_CHAIN WRITE permission, and SHALL NOT
introduce any new permission check for it. Activating it SHALL run the editor's advisory checks over the
graph currently on the canvas and SHALL open the findings drawer with the result.

The action is advisory only: it SHALL NOT alter the graph, SHALL NOT mark the page dirty, and SHALL NOT
affect whether the chain can be saved.

#### Scenario: A read-only viewer opens the editor
- **WHEN** a user whose role lacks RULE_CHAIN WRITE opens the rule-chain editor
- **THEN** the Analyze control is present and enabled, even though the node library, the library toggle, the
  add-note button and the footer FAB section are absent

#### Scenario: A viewer runs the checks
- **WHEN** a read-only viewer activates Analyze
- **THEN** the checks run, the findings drawer opens with the result, and the graph, the dirty state and the
  save control are unchanged

#### Scenario: Analyze is activated twice
- **WHEN** the user changes the canvas and activates Analyze again
- **THEN** the drawer shows the result of the new run, replacing the previous findings

### Requirement: A dismissible findings drawer at the bottom of the canvas
The system SHALL present the findings of a run in a drawer at the bottom of the editor canvas, with one row
per finding, and SHALL make it dismissible by the user. The canvas SHALL keep its full width while the drawer
is open, and the drawer SHALL open and close independently of the node-configuration drawer — either may be
open while the other is open or closed.

Each finding row SHALL show the check type, the name of the node the finding is about, and a short
explanation, and SHALL be styled as a warning, not an error. When a run produces no findings, the drawer
SHALL show exactly the text `No issues found by these checks.` and no rows.

#### Scenario: The drawer opens over a full-width canvas
- **WHEN** the findings drawer opens
- **THEN** it occupies the bottom of the canvas area, and the canvas keeps the same width it had before

#### Scenario: The user dismisses the drawer
- **WHEN** the user dismisses the findings drawer
- **THEN** it closes, the findings are no longer shown, and nothing else about the page changes

#### Scenario: The node-configuration panel is open at the same time
- **WHEN** the findings drawer is open and the user opens a rule node for editing
- **THEN** both are open at once, and closing either one leaves the other open

#### Scenario: A finding is rendered
- **WHEN** a run produces a finding about a node
- **THEN** its row shows the check type, that node's name and a short explanation, with warning styling and
  not error styling

#### Scenario: A run produces no findings
- **WHEN** a run produces no findings
- **THEN** the drawer shows exactly `No issues found by these checks.` and lists no rows

### Requirement: The reachability check reports nodes with no path from the chain input
The system SHALL provide a reachability check that walks the connections of the graph **currently on the
canvas** forward from the chain's input node, and SHALL report every node it does not reach. The check SHALL
read the live, unsaved graph, so that nodes and connections the user has added, edited or deleted without
saving are taken into account.

Each reported node's finding SHALL carry the text `Not reachable from chain input` verbatim, and its
explanation SHALL NOT state or imply that the node can never execute.

The input node itself SHALL never be reported, and canvas notes SHALL never be reported, because they take
no part in the graph. A node that cannot receive input at all is reported like any other unreached node: this
change adds no node-type exclusion.

When the chain has no first node — no connection leaves the input node — the system SHALL report one
chain-level finding that names no node, instead of one finding per node on the canvas.

#### Scenario: A node with a path from the input
- **WHEN** a node is reachable from the input node by following connections forward
- **THEN** the check reports nothing about it

#### Scenario: A node with no path from the input
- **WHEN** a node has no path from the input node
- **THEN** the check reports it with the text `Not reachable from chain input`, and the explanation does not
  claim the node can never execute

#### Scenario: A node is added and left unconnected without saving
- **WHEN** the user drags a new node onto the canvas, connects nothing to it and does not save
- **THEN** the next run reports it as not reachable from chain input

#### Scenario: The only inbound connection is deleted without saving
- **WHEN** the user deletes the only connection leading to a node and does not save
- **THEN** the next run reports that node as not reachable from chain input

#### Scenario: A node that accepts no input
- **WHEN** a node on the canvas can accept no incoming connection, so no forward walk can reach it
- **THEN** it is reported like any other unreached node

#### Scenario: Notes and the input node
- **WHEN** the canvas holds notes and the synthetic input node
- **THEN** neither is ever the subject of a finding

#### Scenario: The chain has no first node
- **WHEN** no connection leaves the input node
- **THEN** the run produces exactly one chain-level finding, which names no node, instead of one finding per
  node on the canvas

### Requirement: Findings are ordered by check type then node name and deduplicated per check
The system SHALL order the findings of a run by check type and then by node name, and SHALL report a given
node at most once per check type, however many times that check's condition applies to it.

#### Scenario: Several nodes are reported by the same check
- **WHEN** a run reports more than one node for the same check
- **THEN** those rows appear consecutively, in node-name order

#### Scenario: One node matches a check in more than one way
- **WHEN** a node satisfies a check's condition by more than one route — for example several missing inbound
  paths
- **THEN** it appears exactly once for that check

## MODIFIED Requirements

### Requirement: ui-ngx has no test runner
The system SHALL provide a `test` script in `ui-ngx/package.json` that runs the module's unit tests, backed
by a minimal Jest setup scoped to **plain TypeScript** sources only — no Angular TestBed, no browser
environment, and no change to the Angular application build or to the Maven build. Tests SHALL be runnable
from `ui-ngx` with `yarn test`, and SHALL pass without a running server or browser.

Within that scope: `ui-ngx/package.json`'s other scripts (`ng`, `start`, `build`, `build:prod`,
`build:types`, `build:icon-metadata`, `lint`, `prepare`) are unchanged; `angular.json` still defines no
`test` target, because the runner is not an Angular builder; and `src/tsconfig.app.json` continues to exclude
`test.ts` and `**/*.spec.ts` from the application compilation, so spec files added under `src` do not enter
the Angular build. Linting stays ESLint 9 with `angular-eslint` through `yarn lint` (`eslint.config.mjs`),
which types every `**/*.ts` with `parserOptions.project: true`, so a spec file must belong to some TypeScript
project and Jest's globals must be declared for it. `ui-ngx/pom.xml` drives the frontend with
`frontend-maven-plugin` (yarn 1.22.22, `pom.xml:61`): `yarn install` (`:65`–`70`) and, in the `yarn-build`
profile, `run build:prod` (`:136`–`141`) — still no test goal, so adding the script does not change the
Maven build. The runner has to mirror the root `tsconfig.json`'s `module: "es2020"` with
`moduleResolution: "node"` and its path aliases (`@app/*`, `@core/*`, `@shared/*`, `@home/*`, plus
file-level aliases for `jquery`, `ace`, `tooltipster`, `jszip`, `jstree`) only as far as the files under test
actually use them; installation still depends on `prepare` running `patch-package --error-on-fail` over the
eight patches in `ui-ngx/patches/`, with `ngx-flowchart` pulled straight from a git tag
(`package.json:88`), which is why the runner deliberately stays away from Angular and flowchart tooling.
Testing an Angular component or template in `ui-ngx` therefore still requires standing up Angular test
tooling, which this setup does not provide.

#### Scenario: Running the UI tests
- **WHEN** `yarn test` is run in `ui-ngx`
- **THEN** the Jest setup runs the module's spec files and reports their result

#### Scenario: A spec file is added under src
- **WHEN** a `*.spec.ts` file is placed under `ui-ngx/src`
- **THEN** `tsconfig.app.json`'s `exclude` keeps it out of `ng build`, and it belongs to a TypeScript project
  that the Jest setup and `eslint.config.mjs` both resolve

#### Scenario: A spec file uses Jest globals
- **WHEN** `describe`/`it`/`expect` appear in a linted file
- **THEN** `yarn lint` accepts them, because `eslint.config.mjs` has a spec-scoped block declaring Jest's
  globals

#### Scenario: The Maven build runs the UI
- **WHEN** `ui-ngx/pom.xml` drives the frontend through `frontend-maven-plugin`
- **THEN** it runs `yarn install` and `yarn build` only and invokes no test goal, so the `test` script does
  not change the Maven build

#### Scenario: An Angular component needs a test
- **WHEN** someone wants to test a component, template or Angular service in `ui-ngx`
- **THEN** this setup does not cover it: Angular test tooling still has to be added first

### Requirement: English UI strings live in one locale constant file
The system SHALL read UI strings through the `translate` pipe against `ui-ngx/src/assets/locale/locale.constant-en_US.json`, a single nested JSON object whose `"rulechain"` section starts at line 7566 and whose `"rulenode"` section starts at line 7640. Keys are dot paths into it, e.g. `{{'rulechain.add-note' | translate}}` (`html:200`), `{{'rulechain.note' | translate}}` (`html:167`), `{{'rulenode.search' | translate}}` (`html:37`). The directory holds 28 locale files (`ar_AE` … `zh_TW`); non-English locales are separate files and untranslated keys fall back. Within `"rulechain"` the keys are not sorted — they are grouped by feature, with the most recently added feature's keys appended last.

The analysis feature's keys are appended to the `"rulechain"` section in the same way, in English only. Two of
their values are fixed by acceptance criteria and SHALL be stored verbatim, with no surrounding markup,
punctuation change or interpolation placeholder: `Not reachable from chain input` and
`No issues found by these checks.` The remaining keys — the Analyze label, the drawer title, the check-type
label, the explanation text, the chain-level no-first-node message and the dismiss tooltip — are ordinary
English strings.

The template uses three translation forms: the `translate` pipe inside attribute interpolation (`matTooltip="{{'rulechain.add-note' | translate}}"`, `html:200`); a ternary key then the pipe for toggles (`{{ (drawer?.opened ? 'rulenode.close-node-library' : 'rulenode.open-node-library') | translate }}`, `html:189`); and the `translate` structural attribute where the element's text *is* the key (`<span translate>{{menuItem.value}}</span>`, `html:264`) — which is how every context-menu label is translated, `RuleChainMenuItem.value` holding a raw key. In TypeScript, `translate.instant(...)` supplies hotkey descriptions (`ts:400`–`499`), context-menu subtitles (`ts:744`) and tooltip bodies (`ts:1526`–`1575`).

#### Scenario: A key is added to English only
- **WHEN** a new key is added under `"rulechain"` in `locale.constant-en_US.json` and nowhere else
- **THEN** English shows the new string and other locales fall back to it

#### Scenario: An acceptance-tested string is read
- **WHEN** the UI renders the reachability finding text or the empty-result text
- **THEN** the rendered text is exactly `Not reachable from chain input` and exactly
  `No issues found by these checks.`, because those are the literal values in the en_US file
