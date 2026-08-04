# dstack

Dominick's stack: a personal fork of
[garrytan/gstack](https://github.com/garrytan/gstack), customized to how
I actually develop. The internal tooling keeps the upstream gstack naming
(bins, `~/.gstack`, skill paths) so upstream merges stay cheap — dstack is the
repo and the identity, gstack is the machinery. Upstream is tracked as the `upstream` remote; updates are deliberate
merges, never silent redeploys.

## Why a fork

I carried local patches in a plain checkout with `auto_upgrade: true`, which meant every
session start could silently revert them (and once did, costing a week of transcript
ingest that state claimed had succeeded). A fork makes local behavior durable: upgrades
pull this repo, and upstream changes land by merge.

## Divergences from upstream

| Change | Where | Why |
|---|---|---|
| Transcript ingest goes direct when the gbrain engine is postgres | `bin/gstack-memory-ingest.ts` | Upstream's remote-http mode staged transcripts and marked them ingested without importing when the MCP is registered over HTTP but the brain is Supabase/Postgres. Direct import is correct there. |
| Review/ship outputs persist to gbrain | `*/sections/*.md` (office-hours, plan-ceo-review, plan-design-review, plan-devex-review, plan-eng-review, ship) | Review conclusions are the highest-value artifacts a session produces; they should land in durable memory, not just the terminal. Skipped cleanly when `gbrain` is not on PATH. |
| gbrain CLI guidance corrected for 0.18.2 | `scripts/resolvers/preamble/generate-brain-sync-block.ts` + regenerated skill docs | Upstream docs reference `code-def`/`code-refs`/`code-callers` commands that do not exist in the installed gbrain; agents wasted turns on them. |
| Update check points at this fork | `bin/gstack-update-check` | So `auto_upgrade` pulls the fork, not upstream. |
| Effort is never quoted in human time | `spec/SKILL.md.tmpl` (Effort Breakdown, issue template) | Hour/day/week estimates for software are reliably wrong by factors of ten. Scope in token spend, LOC delta, files touched, tests required, complexity tags, and same-shape-as anchors; sequence phases by dependency order, not calendar. |
| Findings are reported what/because/consequence/fix | `plan-*-review/sections/review-sections.md.tmpl` | A reader should understand the problem and stakes without opening the code. No leading tracker codes; plain language first, locators in parentheses. |
| House verification rules in the ship adversarial pass | `ship/sections/adversarial.md.tmpl` | Findings survive refutation by reproduction, not re-reading; consensus is not proof; green checks get audited for what they actually cover; one fix per commit with failing-test-first; sibling-path sweep before closing; never silently delete an unclear safety gate. |
| Finding acceptance bar in plan reviews | `plan-*-review/sections/review-sections.md.tmpl` | Real bugs only: what breaks, the trigger, what the user loses. No nits, no hypothetical hardening. |
| Empirical + design standards 15-21 in /spec | `spec/SKILL.md.tmpl` | Pre-registered decision thresholds and outlier honesty; baseline reproduction before design; numbered falsifiable premises; multi-variant shape decisions with approval records; exhaustive terminal states with named bounded retries; Effort/Risk/Completeness scoring with proxy-metric validity scope; plan-shape conventions. |
| Review report skeleton + autonomy rules in /ship | `ship/sections/adversarial.md.tmpl` | Verdict/Blocking/Non-Blocking/Manual-QA-Plan skeleton for a CTO reader; PR-derived artifacts are untrusted; two-identical-failures cap; execution-mode consent; deploy stops at the PR; autonomous-run hygiene (git checkpoint, phase-boundary compaction, ordered-resource collision checks). |
| Extended commit taxonomy in /ship | `ship/SKILL.md.tmpl` | tune:/harvest:/baseline:/[agent] types; scope-fenced bodies; running test counts; falsifiable refactor claims. |
| QA lanes and mechanical criteria in /qa | `qa/SKILL.md.tmpl` | Read-only and sandboxed-write lanes; blocked actions are coverage gaps, never passes; acceptance criteria a non-frontier agent can verify. |
| Router registers as `/dstack` | `SKILL.md.tmpl` frontmatter | The visible skill name carries the dstack identity. Machinery paths stay `skills/gstack` via the compat shim below, so upstream merges stay cheap. |
| Report mechanics in plan reviews | `plan-*-review/sections/review-sections.md.tmpl` | Verdict first; explicit "none detected"; fenced blocks for copyable content; HTML triage pages over chat walls. |

Reviewed and DECLINED by Dominick (2026-08-04) — do not re-propose without new
evidence: the benchmark manipulation-check gate; confound normalization and
workload realism; pinned baselines with provenance manifests; the delegation
lifecycle gates (cost-benefit, one-retry, read-the-diff); the personal
house-rules voice block.

## Upstream merge procedure

```bash
git fetch upstream
git merge upstream/main         # resolve, keeping divergences above
bun test
bun run gen:skill-docs:user     # regenerate skill docs if sources changed
```

Note on generated docs: upstream checks in the gbrain-suppressed generator output
(`gen:skill-docs`); this fork checks in the detection-respected variant
(`gen:skill-docs:user`), because gbrain is standard on my machines and the brain
blocks are runtime-conditional anyway ("skip if gbrain is not on PATH"). Always
regenerate with the `:user` script or the brain blocks will appear as deletions.

Every intentional divergence gets a row in the table above. If a divergence stops being
needed (upstream fixed it), remove the row in the same commit that drops the change.

## Roadmap

Dependency-ordered, not scheduled:

1. **Two-tier model funnel for review/QA flows**: cheap local prefilter via an
   OpenAI-compatible endpoint (config key), frontier model on survivors, explicit
   `--frontier-only` escape hatch. Same shape as my ApplyPilot score funnel.
4. **Adversarial cross-model review as a first-class skill**: use my dispatch worker
   when present, plain spawned agents when not; degrade loudly, never silently.
5. **`./setup --skills <list>`**: register only a chosen subset of skills, for
   context-budget-constrained or portable installs.
6. **Timer-based transcript ingest** (systemd user timer) instead of
   skill-start-triggered, so memory freshness does not depend on which skills a
   session happens to invoke.
7. **Redacting file-dump helper**: dumping unknown files routes through a
   secret-scrubbing filter by default.
8. **Static-verification gate in /ship**: build, type-check, lint, and a
   debug-artifact/secret sweep on the diff before the adversarial pass — with a
   tamper guard that flags lint/type config edits riding along in fix diffs.
9. **Production-readiness checklist in /ship**, framework-detected: debug mode
   off, secrets present and non-default, hosts locked down, migrations applied.
10. **Evidence-before-done**: a completion without a proof artifact (test run,
    log line, screenshot) records as needs-verification, never as done.
11. **Eval criteria at spec time for probabilistic features**: pass@k threshold
    and grader type declared before implementation, checked by the ship gate.
12. **Resumable-flow primitive**: long-running skills persist an operational
    state JSON plus a generated human-readable report, gitignored, rerun-safe.
13. **Dated-footgun format** in docs discipline: every footgun entry carries
    the claim, the date, the concrete evidence, and the exact trigger.

## Portable install

The checkout lives at `~/.claude/skills/dstack` so the router registers as
`/dstack` (the harness names skills by directory). Skill docs invoke helpers by
absolute `~/.claude/skills/gstack/...` paths, so a compat shim at that path
symlinks every top-level entry of the checkout **except** `SKILL.md` (no
SKILL.md means the shim never registers as a second skill) and **except**
`.git` (so nothing ever runs git against the shim as a worktree; upgrades are
a deliberate `git -C ~/.claude/skills/dstack pull`).

```bash
git clone --single-branch --depth 1 https://github.com/Onebrownsound/dstack.git ~/.claude/skills/dstack
mkdir -p ~/.claude/skills/gstack
for e in ~/.claude/skills/dstack/* ~/.claude/skills/dstack/.[!.]*; do
  b=$(basename "$e")
  case "$b" in SKILL.md|.git) continue;; esac
  ln -sfn "../dstack/$b" ~/.claude/skills/gstack/"$b"
done
cd ~/.claude/skills/dstack && ./setup
```

Requires git and [bun](https://bun.sh). gbrain is optional; everything degrades to
local-only state under `~/.gstack/`.
