<!--
template: agents-review-entrypoint
version: 1.1.0
source-of-truth: t0t005081114-hue/otomo-core harness/templates/agents-review-entrypoint.md
guidance: harness/CODEX_REVIEW.md
Copy to the repository root as AGENTS.md and fill in the <...> placeholders.
Delete this comment block and any line that does not apply. Do not list documents that do not exist.
Keep the file a router: link to Source of Truth, do not restate it. Put only repository-specific review deltas in this file.
-->

# AGENTS.md

This file is the **Codex Review Entry Point** for `<repository>`. It tells Codex what to read, where each review rule is owned, and what is specific to reviewing this repository. It is a router, not a Source of Truth: it does not define or override any rule in the documents it links to.

`CLAUDE.md` is the Claude Code entrypoint for implementation / operation. This file governs Codex review. Shared rules are owned by the Source of Truth and the OTOMO CORE Harness, not by either entrypoint.

## Repository Context

- Role: <Product repository | Lead operation repository (not a Product)>
- <one or two lines on what this repository owns>
- Judge against the formal Source of Truth listed below. GitHub evidence outranks implementer reports and chat summaries.

## Required Context Before Review

Read before forming a verdict. List only documents that exist in this repository.

Remote Review adopters: describe the file-backed required documents in a form the Product's Remote Review implementation can parse. The concrete format follows Product-side ownership (`harness/REMOTE_REVIEW.md` §3, §9). This template's numbered list is not a parser-compatible format.

1. <requirements>
2. <formal specification>
3. <Acceptance Criteria / implementation plan>
4. <Product- or Lead-specific rules>
5. <decisions / failures / phase records>
6. Recent relevant PRs and prior review findings for this change, and unresolved blocking issues / Human Decision Required items
7. OTOMO CORE Harness (sibling checkout `..\otomo-core`) — **native review context only**. Read it from the approved CORE baseline as defined in `harness/DEVELOPMENT_STANDARDS.md` §7 Approved CORE Baseline (native application: `harness/CODEX_REVIEW.md` §6). Bootstrap, so you can reach §7 itself safely (§7 stays the authority for everything else):
   - The default approved ref is `origin/main`. When using it, run `git -C ..\otomo-core fetch origin` first.
   - At review start, resolve the approved ref **once** to a commit SHA (e.g. `git -C ..\otomo-core rev-parse 'origin/main^{commit}'`; keep the quotes in PowerShell) and read CORE documents only from that SHA for the whole review (`git -C ..\otomo-core show <resolved-SHA>:<path>`). Do not use the checked-out branch or working tree as the baseline.
   - If fetch is not possible, only a Human-approved SHA may be used. If the approved baseline cannot be established, do not substitute another revision: use only the documented fallback below, otherwise fail closed.

   These files are outside this checkout and are not file-backed Source of Truth for Remote Review; do not add them to the Remote Review context manifest (`harness/CODEX_REVIEW.md` §6):
   - `harness/CODEX_REVIEW.md` — native review entry point
   - `harness/DEVELOPMENT_STANDARDS.md`
   - `harness/PHASE_WORKFLOW.md`
   - `architecture/RESPONSIBILITY_BOUNDARIES.md`
   - <Lead repositories: `architecture/LEAD_AGENTS.md`>

If the approved CORE baseline cannot be read, resolved, or confirmed, follow `harness/DEVELOPMENT_STANDARDS.md` §7 Approved CORE Baseline (documented fallback only; otherwise fail closed). Documented fallback for this repository: <name it, or delete this line if none exists>.

Rule precedence is defined by OTOMO CORE (`architecture/RESPONSIBILITY_BOUNDARIES.md` §5; Lead repositories also `architecture/LEAD_AGENTS.md` §19). Listing a document here gives it no extra authority. Conflicts are reported with their precedence level, not resolved by the reviewer (`harness/CODEX_REVIEW.md` §3.3).

## Review Routing

Review rules are owned by the OTOMO CORE Harness (read from the approved CORE baseline) and this repository's Source of Truth. This file does not restate them.

- General review policy — review criteria, TBD handling, Review Handoff and scope, Review Assurance Level, Finding and severity (blocking / advisory), Durable History, Evidence Integrity, reviewer permissions: `harness/DEVELOPMENT_STANDARDS.md` (§1–§5)
- Review lifecycle — review unit, remediation, re-review scope, Finding Trace, New Finding Stop Rule, completion: `harness/PHASE_WORKFLOW.md` (§4–§6)
- Native Codex mechanics — effective instruction chain, changed-path instruction applicability, instruction-source changes and trusted baseline, self-reference residual risk: `harness/CODEX_REVIEW.md` (§3, §6.1)
- Reviewer conduct, topic by topic, with the owning section of each rule: `harness/CODEX_REVIEW.md` §4
- Responsibility boundaries and precedence: `architecture/RESPONSIBILITY_BOUNDARIES.md` <Lead repositories: and `architecture/LEAD_AGENTS.md`; Lead review model: `harness/CODEX_REVIEW.md` §7>
- If the diff adds, modifies, or removes a repository-local Codex instruction source (including this file), apply `harness/CODEX_REVIEW.md` §3.2 and §6.1 before forming a verdict.
- Never output secrets, credentials, tokens, personal data, or environment variable values (`harness/DEVELOPMENT_STANDARDS.md` §5 Evidence Integrity).

## Repository-specific Review Focus

<Optional: repository-specific review focus / deltas only, one line per item, linking to the repository rule that defines it. Do not restate CORE rules here.>
