---
id: plans/2026-10-23-pi-renderer-network-read-mcp
title: "Pi renderer: lower network.read to the built-in mcp tool (eight-tool vocabulary)"
doc-type: plan
status: draft
component: KyberSquad
owner: dpalfery
last-reviewed: 2026-10-23
development-mode: test-first
autonomy: autonomous
---

# Pi renderer: lower network.read to the built-in `mcp` tool (eight-tool vocabulary)

**Status: Ready — decision-complete. D1 decided by the owner (Option A, relayed by the
conductor 2026-10-23); D2–D6 decided under `autonomy: autonomous` with provenance recorded
per row. `docs validate` / `docs drift` could not be run in the authoring harness (no
process tool; G3, same as the 2026-10-23 opencode-context plan) — the conductor runs both
before execution.** Frontmatter `status: draft` per closed ontology — body Ready is
authoritative.

**Development mode:** `test-first` (default; no opt-out supplied).
**Autonomy:** `autonomous` (resolved from repository root `AGENTS.md` Config Reg `<autonomy>`, confirmed by the packet).

## 1. Problem and goal

### 1.1 Defect

The Pi renderer's closed `tools` vocabulary deliberately maps only four capabilities
(`filesystem.read`, `filesystem.search`, `filesystem.write`, `process.execute`) onto seven
built-in tool names, and leaves `network.read` / `network.publish` unmapped on the premise
that Pi has no built-in tool for either (plan `2026-09-14-pi-harness-target` R8, owner
decision Q1-A 2026-09-14). The consequence: **`research-agent` — whose canonical profile
grants `network.read: allow` and whose body tells it to prefer MCP tools — renders on Pi
with `tools: read, grep, find, ls` and no web access at all**, while the renderer honestly
records `permission-not-expressible` (the release knew and accepted this).

That premise is now false for `network.read`. Live verification this session established:

- `mcp` and `bash` are first-class harness tool names (the main session used both).
- A locally hand-patched `~/.pi/agent/agents/research-agent.md` whose `tools` list carried
  `bash, mcp` was accepted by the pi-subagents loader and restored full function —
  including a sourced npm-tarbell registry lookup and a successful `bash`/`curl` call
  against `registry.npmjs.org`.

So `mcp` is an expressible Pi tool name for read-oriented network access, and the current
mapping silently drops a capability the canonical profile grants.

### 1.2 Goal

Lower `network.read: allow` onto the built-in `mcp` tool in `PiRenderer` (owner decision
**Option A**): add `mcp` to the capability→tool map and the fixed `ToolOrder`, and stop
recording `permission-not-expressible` for `network.read`. Everything else about the Q1-A
envelope is preserved: `process.execute` stays withheld for read-only roles (**no shell for
read-only**), `network.publish` stays unmapped and still records `permission-not-expressible`,
`extensions: false` is still emitted unconditionally, and the `tools` key is still always
emitted. `PiRendererContractTests` is updated with the new mapping — strengthened, not
weakened. The unverified remainder (MCP-grant *enforcement* inside a subagent carrying
`mcp` under `extensions: false`) is sequenced as an explicit fail-closed live check before
closeout.

## 2. Decisions

| ID | Question | Answer / decision | Mitigating and supporting information | Status / approval provenance |
|---|---|---|---|---|
| D1 | Fix approach for the dropped `network.read` grant on Pi | **Option A (minimal):** add `mcp` to the Pi tool vocabulary and `ToolOrder`; map `network.read: allow → mcp`. `process.execute: deny` stays withheld for read-only roles — no shell for read-only. | Matches the research-agent body's "prefer MCP tools" instruction and the canonical `read-only` profile's intent (web access without a shell). Option B rejected on least-privilege grounds: mapping `network.read → bash` as well, or moving research-agent to the `investigator` profile (`process.execute: allow`), hands a read-only role a full shell — a strictly larger grant than the capability it expresses, and `bash` is already reachable on Pi through `process.execute` for the roles that legitimately need it. Option B is not weighed further per the owner's direction. | ANSWERED — owner decision relayed by conductor 2026-10-23 ("OPTION A … do not spend further effort weighing option B"). |
| D2 | Revisit of the Q1-A precedent (2026-09-14, `pi-subagents` 0.19.0 README basis, owner decision recorded in plan `docs/archive/plans/2026-09-14-pi-harness-target.md` and ADR 0019's envelope) | Split revision: only the **R8 vocabulary fact** is superseded — `network.read` now has an expressible built-in tool (`mcp`, live-verified this session against the installed harness). The rest of Q1-A stands unchanged: `extensions: false` unconditionally, always-emitted `tools`, `ask`/`deny` both withheld, `network.publish` still unmapped. | The R8 premise ("neither has a built-in Pi tool") was fact-based, not preference-based; new live evidence invalidates it for `network.read` only. Honoring the accepted decision as far as it remains true (autonomous-decisions check 3): the extension-loading decision (Q1-A proper) is untouched, so no conflict with the owner's original intent remains. | ANSWERED — autonomous: recommendation accepted; covered by owner-supplied live evidence and the Option A owner decision (D1). |
| D3 | Record for the durable decision change | Amend [ADR 0019](../adr/0019-pi-native-subagents-and-primary-lowering.md) **in place** with a dated amendment recording the `network.read → mcp` lowering, the live-evidence basis, and the explicit statement that `network.publish` remains `permission-not-expressible`. No new ADR. | Repo precedent for refinements of an existing decision: ADRs 0005, 0016, 0031 amended in place. This is a refinement of the Pi native rendering contract's tool vocabulary, not a new boundary, dependency, schema, or target. A new ADR would split the Pi rendering record across two documents for no gain. | ANSWERED — autonomous: recommendation accepted; established amend-in-place pattern. |
| D4 | Placement of `mcp` in the fixed `ToolOrder` CSV | Append at the end: `read, grep, find, ls, edit, write, bash, mcp`. | The order is a byte-stability contract, not a semantic ranking; any fixed position is conformant. Appending minimizes diff churn in the emitted CSV for agents that do not hold `network.read`, keeping rendered files byte-stable wherever the capability set is unchanged. Cheap to reverse. | ANSWERED — autonomous: low-confidence, no authoritative source (cosmetic ordering choice with a stated reason). |
| D5 | Deployed `~/.pi/agent/agents/*.md` renders in the wild (rendered under the old mapping) | Next `squad update` / `squad install --target pi` re-renders and **overwrites** the receipt-managed agent files with the new mapping — expected. Local manual patches are superseded; on this machine the hand-patched `research-agent.md` (bash + mcp) loses its `bash` again (`process.execute: deny` restored for read-only) while `mcp` arrives canonically. Machines that never re-run squad update keep their old renders until their next update; there is no remote push. | ADR 0037 receipt-managed deployment; the local patch's purpose (web access) is now served canonically, so superseding it is the fix, not a regression. If `squad update` reports drift on the locally modified managed file, accept the overwrite. Owner call-out recorded in §9 so the surprise is pre-announced. | ANSWERED — autonomous: recommendation accepted; consequence of ADR 0037 deployment mechanics, relayed to the owner in §9. |
| D6 | MCP-grant enforcement inside a subagent carrying `mcp` in `tools` (under `extensions: false`) is NOT yet verified | Sequence it as a **fail-closed live check (G2 / T4)** before closeout: deploy the new render locally and exercise a read-only subagent's access to an MCP tool. If MCP is unreachable, the mapping is inert; T4 fails closed and the emission question (the `extensions` key — itself Q1-A territory) goes back to the conductor/owner. No developer improvises an `extensions` change. | The live patched file proved the loader *accepts* `mcp`; it did not prove an MCP tool *answers* under `extensions: false` (the patched session's registry access went through `bash`/`curl`). This is the one load-bearing unknown in Option A; a silent ship would repeat the original defect's failure mode (accepted-but-dead tool name) under a different mask. | ANSWERED — autonomous: recommendation accepted; packet instruction "sequence that as a fail-closed check" with contingency defined. |

## 3. Investigation findings

**Discovery report:** `Discovery: codegraph 0, docs 0, direct reads 16 (self-gathered)`.
The harness exposes no `codegraph_explore` and no Kyber-Weave docs MCP tools; per the
fallback rule, discovery used the documentation index (`docs/plans/README.md`,
`docs/catalog.md`, Config Reg in root `AGENTS.md`) plus narrow direct reads, labeled
self-gathered. Direct reads of indexed code (all narrow, with reason): `PiRenderer.cs`
regions :40–140, :450–640 (pinned evidence: vocabulary doc, `CapabilityTools`, `ToolOrder`,
`ResolveTools`, the not-expressible degradation branches, the `tools`/`extensions` emission
at :326–331) — reason: the defect site; `PiRendererContractTests.cs` regions :1–100,
:300–370, :560–640 — reason: the contract pin named in the packet; `capabilities.yml` (all
profiles) and `research-agent.md` — reason: the profile/body evidence named in the packet;
`grep` over `products/kyber-squad/agents/*.md` for `capability-profile` — reason: blast
radius. Documentation reads: plan index, one active-plan frontmatter precedent, one archived
Squad plan precedent, ADR 0019, `requirements.md` target table, `onboarding.md` Pi notes,
`catalog.md`, the `kyber-weave-docs` skill contract.

Key facts established:

1. **Defect site** — `src/KyberWeave.Core/Squad/Rendering/PiRenderer.cs`: the seven-tool
   vocabulary is documented at :59–69 (owner decision Q1-A 2026-09-14, `pi-subagents`
   0.19.0 README basis); `CapabilityTools` (the 4-entry map, `network.read`/`publish`
   deliberately unmapped) and `ToolOrder` at :117–133; `ResolveTools` at :469–494
   (allow-only, non-broadening by construction); `BuildSubagentDegradations` at :533–544
   records `permission-not-expressible` for allowed `network.read` and `network.publish`
   (plus the empty-roster `delegate` case). The `tools` frontmatter emission at :326–328
   (always emitted, literal `none` when empty) and `extensions: false` at :330–331 are
   unaffected by this change.
2. **Canonical grant** — `products/kyber-squad/profiles/capabilities.yml`: `read-only`
   grants `network.read: allow`, `process.execute: deny`, `delegate: deny`; `network.publish`
   is `allow` on **no** profile (`publishing-worker` holds `ask`), so after this change no
   subagent profile yields a `network.publish` not-expressible record — only the empty-roster
   and primary-lowering cases remain, and the combined not-expressible set stays non-empty
   (conductor's primary fallback always records it).
3. **Body evidence** — `products/kyber-squad/agents/research-agent.md`: description and
   Method step 2 ("Prefer the dedicated MCP tool for a domain when one is connected"), and
   `copilot-tools` already include `web`, `search`, `context7/*` — the canonical body assumes
   web+MCP; only the Pi lowering drops it. No `products/` file needs editing.
4. **Blast radius (subagent agents whose profile grants `network.read: allow`)** — profiles
   `architect`, `read-only`, `reviewer`, `investigator`, `publishing-worker`: agents
   `architect`, `research-agent`, `azure-reader`, `review-lens`, `review-triage`,
   `code-reviewer`, `bug-crusher-investigator`, `github-devops`. Each gains `mcp` in its
   rendered `tools` CSV. `worker`/`root-fixer`/`documentation`/`orchestrator` profiles deny
   `network.read` — their renders are byte-identical to today (`ToolOrder` appends, it does
   not reorder). `conductor` (orchestrator, primary) is unaffected in its lowered-skill path.
   `capability-not-isolable` is untouched: `mcp` grants no shell, so
   `grantedShellTools: ["bash"]` semantics are unchanged.
5. **Contract pin** — `tests/KyberWeave.Tests/PiRendererContractTests.cs`: independent
   transcription `CapabilityToolContract` at :45–57 (comment :47 cites R8),
   `ToolOrder` replica at :61, per-agent `tools` assertions at :317–328, key order
   `name, description, [model], [thinking], tools, extensions, [allowed_subagents]` at
   :346–363, and the generic not-expressible predicate at :588–637 (comment :588 cites
   R9/R10). No test asserts the not-expressible `Details` prose, so removing the
   `network.read` branch breaks no pinned string.
6. **Live evidence (this session, owner-supplied)** — `mcp` and `bash` are first-class
   harness tools; a hand-patched agent file whose `tools` carried `bash, mcp` was accepted
   by the loader and restored full research-agent function (bash/curl against
   `registry.npmjs.org` verified). MCP-grant *enforcement* in a subagent carrying `mcp`
   under `extensions: false` remains unverified → D6/T4.
7. **Precedent pattern** — mapping `network.read` onto a target's native tool name is the
   established per-target lowering (ZCode → `WebFetch`/`WebSearch`; OpenCode →
   `webfetch`/`websearch`; Devin → documented web tools per ADR 0025). Option A brings Pi
   into that pattern rather than inventing a new mechanism.
8. **Doc surface** — `docs/kyber-squad/requirements.md` target table Pi row (currently
   "Native execution + safety-narrowed", silent on the vocabulary), `docs/kyber-squad/
   onboarding.md` Pi notes (:216 prerequisite paragraph), and ADR 0019 (no capability-mapping
   text exists in any of them today — the mapping facts live in the archived 2026-09-14 plan
   and the renderer/test remarks). ADR 0019 records native subagents + primary lowering,
   making it the right amend-in-place home (D3).

## 4. Test contract (development-mode: test-first)

| Task | Test project or file | Runner command | Observable behavior | RED evidence required | GREEN acceptance |
|---|---|---|---|---|---|
| T1 | `tests/KyberWeave.Tests/PiRendererContractTests.cs` | `dotnet test tests/KyberWeave.Tests/KyberWeave.Tests.csproj -c Release --no-build --filter "FullyQualifiedName~PiRendererContractTests"` | (a) A `network.read: allow` profile yields `mcp` in the rendered `tools` CSV, in the fixed order; (b) `network.read: allow` produces **no** `permission-not-expressible` record, while `network.publish: allow` and the empty-roster `delegate` cases still do; (c) an explicit named regression pin: `research-agent`'s rendered frontmatter `tools` is exactly `read, grep, find, ls, mcp` — with `bash` absent, proving the read-only no-shell boundary survived the change; (d) the combined expected `permission-not-expressible` set is still asserted non-empty (primary fallback records) — the assertion is preserved, not deleted. | The updated suite fails against the unchanged renderer: the replica lowering omits `mcp` for `network.read: allow` agents (contract-table mismatch and/or the explicit research-agent CSV pin fails), and the not-expressible predicate still yields research-agent where the new expectation excludes it. Recorded in the task digest before T2 starts. | Same suite passes with the assertion set intact and no weakened expectation: `CapabilityToolContract` includes `("network.read", ["mcp"])`, `ToolOrder` includes `mcp`, and every prior assertion (key order, `extensions: false`, `none` literal, degradations) still holds. |
| T2 | No separate test file — T1's contract suite is the test surface; the whole .NET suite is the regression net. | `dotnet test tests/KyberWeave.Tests/KyberWeave.Tests.csproj -c Release --no-build` | Renderer emits the eight-tool vocabulary lowering per D1/D4. | RED already established by T1 (RED-then-GREEN sequencing within one contract surface). | Full suite green (T1 suite plus all Squad renderer/canonical-content suites), `TreatWarningsAsErrors` + `AnalysisMode=all` build clean. |
| T3 | Documentation edits — verification is `docs validate . --merge-ready` and `docs drift .` (read-only checks over the edited corpus). | `dotnet run --project src/KyberWeave.Cli --no-build -c Release -- docs validate . --merge-ready` and `... docs drift .` | ADR 0019 amendment and the requirements/onboarding rows state the `network.read → mcp` lowering, the `network.publish` residual, and the Q1-A partial-revisit — no doc still claims `network.read` is not expressible on Pi. | Genuinely no-unit-test task (documentation only): the read-only verification that replaces it is both docs checks returning zero findings, plus the corpus-wide grep proving no surviving "seven-tool"/R8-style claim for `network.read`. Recorded in the task digest. | Both checks zero findings; grep clean. |
| T4 | Live harness (no unit-test surface exists for subagent MCP enforcement). | Manual/scripted live check in a Pi session with `@tintinweb/pi-subagents` installed: deploy the new render (local `squad update --global` or `squad install --target pi --global`), spawn a read-only subagent (`research-agent`), and have it answer a question that requires an MCP tool. | The subagent carrying `tools: read, grep, find, ls, mcp` (with `extensions: false`) can actually invoke an MCP tool and return sourced content; if it cannot, T4 fails closed. | No RED state applicable (live gate, not a unit contract); the fail-closed requirement is the RED-equivalent: the task may not be reported passed without evidence, and a negative result is a finding, not a shrug. | Evidence recorded in the completion summary: which MCP tool was invoked, from which subagent, under which rendered file. On failure: task returns BLOCKED with the evidence; the `extensions` emission question (Q1-A territory) goes back to the conductor/owner per D6 — no improvised fix. |

## 5. Tasks

### T1 — Contract test update (RED)

- **Objective:** Update `PiRendererContractTests` to pin the new mapping before the renderer
  changes, per the Test contract row above.
- **Files/symbols:** `tests/KyberWeave.Tests/PiRendererContractTests.cs` — `CapabilityToolContract` (:45–57, add `("network.read", ["mcp"])`, rewrite the :45–50 comment: R8 superseded for `network.read` by owner decision 2026-10-23, `network.publish` still unmapped), `ToolOrder` (:61, append `"mcp"`), remarks :31–37 (seven→eight-tool vocabulary; note re-verification 2026-10-23 live session), not-expressible predicate :588–637 (drop `networkReadAllowed` from `expectedNonPrimaryPermissionNotExpressible`; update the :588 comment to name `network.publish` + empty-roster as the remaining R9/R10 cases), and add the explicit `research-agent` `tools`-CSV regression pin (observable `read, grep, find, ls, mcp`).
- **Acceptance:** RED evidence captured (suite fails against unchanged renderer, per T1's Test-contract row); no existing assertion deleted or weakened (the `Assert.NotEmpty` on the combined not-expressible set stays).
- **Dependencies:** none (disjoint from T2's file scope; sequenced RED-first).
- **Required skills:** `test-dev`; the repo's test coding standard at **<test-coding-standard>**.

### T2 — Renderer mapping change (GREEN)

- **Objective:** Implement Option A in `PiRenderer`.
- **Files/symbols:** `src/KyberWeave.Core/Squad/Rendering/PiRenderer.cs` — `CapabilityTools` (:117–128, add `("network.read", ["mcp"])`), `ToolOrder` (:133, append `"mcp"` → `read, grep, find, ls, edit, write, bash, mcp`), `BuildSubagentDegradations` (:533–544, delete the `network.read` not-expressible branch; keep `network.publish` and the empty-roster `delegate` branches), and rewrite the vocabulary remarks (:59–69): eight-tool vocabulary; `network.read` lowers to `mcp` (live-verified 2026-10-23; owner decision Option A; R8/Q1-A 2026-09-14 revisited per D2 — `network.publish` still has no built-in tool; `extensions: false` unchanged).
- **Acceptance:** T1's suite green; full .NET suite green; Release build clean under `TreatWarningsAsErrors` + `AnalysisMode=all`; the "why" comments preserved in house style (the `<remarks>` records the revisit, not just the new fact).
- **Dependencies:** T1 (consumes its RED contract).
- **Required skills:** `csharp-dev`; **<csharp-coding-standard>**.

### T3 — Canonical documentation and ADR amendment

- **Objective:** Bring the corpus current with the changed contract per D2/D3.
- **Files/symbols:** `docs/adr/0019-pi-native-subagents-and-primary-lowering.md` (dated amendment: `network.read → mcp`, live-evidence basis, D2 scope, `network.publish` residual, `extensions: false` unchanged); `docs/kyber-squad/requirements.md` (Pi target-table row: add the `network.read → mcp` lowering and the `network.publish`/lowered-primary `permission-not-expressible` residuals); `docs/kyber-squad/onboarding.md` (Pi notes: read-only roles receive `mcp`; the D5 overwrite consequence in one sentence). Update `last-reviewed` dates on edited docs per the corpus convention.
- **Acceptance:** `docs validate . --merge-ready` and `docs drift .` zero findings (conductor runs them if the executing harness lacks the CLI build); corpus grep shows no surviving claim that `network.read` is not expressible on Pi.
- **Dependencies:** T2 (docs describe the final emitted behavior).
- **Required skills:** `kyber-weave-docs` (frontmatter/validate contract), `app-docs-standard` conventions for the edited app docs.

### T4 — Live fail-closed MCP-grant verification (G2)

- **Objective:** Verify, not assume, that the deployed mapping actually grants MCP tool
  access in a read-only subagent, per D6.
- **Procedure:** Deploy the new render (local `squad update --global` — see §9 for the
  overwrite consequence); from a live Pi session spawn `research-agent`; ask a question that
  requires an MCP tool (e.g., a Kyber-Weave docs MCP lookup or an npm registry lookup);
  confirm the tool answers and the sourced finding returns. The locally observed
  old-patch behavior (bash/curl against `registry.npmjs.org`) is **not** acceptable
  evidence for this gate — the point is the `mcp` grant specifically.
- **Acceptance / fail-closed:** Evidence (tool invoked, subagent, rendered file) recorded in
  the completion summary. On failure: return `STATUS: BLOCKED` with evidence; the `extensions`
  emission question goes to the conductor/owner (D6 contingency) — the mapping is inert and
  the defect is not fixed, whatever the unit suite says.
- **Dependencies:** T2. May run in parallel with T3.
- **Required skills:** any live Pi session able to spawn the subagent (implementing session
  or owner); if only the owner can run it, record `[NOT LIVE-VERIFIED]` per repo precedent
  and leave the gate open — do not claim it.

### Closeout (`docs-dev`)

Standard corpus closeout per the plan-index policy: decision harvest mapping in the
completion summary (D1–D6 → ADR 0019 amendment + requirements/onboarding rows; this plan's
own Decisions table is permanent and preserved), archive per `KW-DOC-LIFECYCLE-003` when the
finishing PR merges.

## 6. Dependency graph

```text
T1 (RED, tests) ──► T2 (GREEN, renderer) ──► T3 (docs + ADR)  ──► closeout
                                            └► T4 (live gate) ──┘
```

T3 and T4 are independent of each other (different file scopes, no consumed output); both
gate closeout. T1's RED evidence must exist before T2 starts (test-first sequencing).

## 7. Risks

| Risk | Mitigation |
|---|---|
| G2 fails: `mcp` is accepted by the loader but MCP tools are unreachable under `extensions: false` — the mapping is inert, silently repeating the original defect's failure mode. | T4 is fail-closed by contract (D6): negative result blocks closeout, evidence goes to the conductor/owner; the `extensions` question is Q1-A territory and is not improvised by a developer. |
| Every `network.read: allow` agent's rendered `tools` CSV changes — receipts and deployed renders shift. | Expected and desired (it is the fix). `ToolOrder` appends without reordering, so all other renders stay byte-identical; receipt-managed update overwrites cleanly (D5). |
| T1's `Assert.NotEmpty` on the combined `permission-not-expressible` set could hollow out if non-primary cases vanish. | Combined set still includes the lowered primary (`conductor`) records; the assertion is preserved as written (Test contract row (d)). If a future corpus change empties it, that is a separate plan — not this one's weakening. |
| Vocabulary doc/test remarks cite `pi-subagents` 0.19.0 / pi-coding-agent 0.84.4 (verified 2026-09-14); the live harness may be newer. | The 2026-10-23 live session re-verified `mcp` acceptance against the *installed* harness; T2's rewritten remarks state both the original pin and the re-verification rather than silently replacing one date with another. |
| `docs validate` / `docs drift` cannot be run in the authoring harness. | Conductor runs both before execution and again at closeout (G3, same disclosure as the 2026-10-23 opencode-context plan). |

## 8. Out of scope

- `network.publish` mapping — still no built-in Pi tool; still records `permission-not-expressible` (D2).
- No changes to `products/kyber-squad/**` — the `read-only` profile and the `research-agent` body are already correct; the defect is purely in the Pi lowering.
- No changes to other renderers, `capabilities.yml`, models, or the deployment/transaction machinery; no `squad doctor` addition for Pi; no new dependencies (Core stays at Markdig + YamlDotNet).
- No `KW-*` rule-id changes (ids are permanent; none are added or renumbered).
- No handling of Option B (rejected in D1; not weighed further per the owner's direction).

## 9. In-the-wild deployment consequence (owner call-out, per D5)

Deployed `~/.pi/agent/agents/*.md` renders in the wild were produced under the old mapping.
On the next `squad update` / `squad install --target pi`, every receipt-managed agent file is
re-rendered with the new mapping and **overwritten** — that is the delivery mechanism for
this fix, not a side effect.

Specifically on this machine: the hand-patched `~/.pi/agent/agents/research-agent.md`
(`tools` carrying `bash, mcp`) is a manual edit of a receipt-managed file. The next update
supersedes it: `bash` disappears again — `process.execute: deny` for the `read-only`
profile is restored, which is the least-privilege boundary Option A deliberately keeps —
and `mcp` arrives through the canonical mapping instead of the patch. If `squad update`
reports drift on the locally modified managed file, accept the overwrite; the patch's
purpose (web access without a shell) is now served canonically.

Machines that do not re-run squad update keep their old (web-less) renders until their next
update; there is no remote push mechanism, and none is added by this plan.

## 10. Verification gates

1. `dotnet format KyberWeave.sln whitespace --verify-no-changes --no-restore -v minimal` and the style pass.
2. `dotnet build KyberWeave.sln -c Release` (warnings-as-errors, `AnalysisMode=all`) and the full `dotnet test` suite.
3. `docs validate . --merge-ready` and `docs drift .` (editing `docs/` and ADR 0019 makes both mandatory; conductor runs them if the executing harness cannot).
4. **`./scripts/update-loop.sh`** — the Squad release path is touched (`PiRenderer` ships in the CLI), so the local release verification loop is required per root `AGENTS.md` / distribution.md. **Note: it needs `node` and `npm` on `PATH`** (it builds `kyberdash` too). It installs no tray and does not run `install.sh`.
5. G2/T4 — the live fail-closed MCP-grant check (§5, D6).
6. Per-task `task-gate` run by each developer task before handoff (repo convention).
7. Whole-change review (`review gates . --out artifacts/gates.json`) before the finishing PR; the PR that finishes this plan archives it (`docs validate --merge-ready` fails on open plans per `KW-DOC-LIFECYCLE-003`).

## 11. Review and closeout

Whole-change review per repo policy (deterministic gate suite plus council). Closeout
(`docs-dev`): harvest mapping — D1/D2/D6 → the ADR 0019 amendment; D1 blast radius and the
residual `network.publish` fact → the `requirements.md`/`onboarding.md` rows; D5 → §9 above
retained in the archived plan for the deployment record. This plan's `## Decisions` table
survives archival intact.
