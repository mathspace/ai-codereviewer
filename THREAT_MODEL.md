# ai-codereviewer Threat Model

## 1. Overview

A public GitHub Action reads a pull-request event, fetches PR metadata/diffs from GitHub, sends selected chunks to OpenAI, and publishes generated inline review comments. Its runtime entry is the checked-in dist/index.js bundle, whose application functions were inspected without execution and agree with the relevant TypeScript flow. This repository’s own review workflow invokes a separately pinned external action, so the reusable local action and repository self-review workflow are distinct deployment paths. No private organization context is incorporated into this model. (`action.yml:18`, `src/main.ts:26`, `dist/index.js:108`, `.github/workflows/code_review.yml:14`)

| Component | Source |
| --- | --- |
| Action interface and bundled runtime | action.yml:3; action.yml:18; dist/index.js:189 |
| GitHub data retrieval | src/main.ts:26; src/main.ts:184 |
| Model prompt/response boundary | src/main.ts:81; src/main.ts:113 |
| Review publication | src/main.ts:149; src/main.ts:169 |

| Deployment or workflow | Resource or capability | Configuration and precedence | Safe effective value or location | Readers, writers, or recipients | Enforcing control | Evidence or unknowns |
| --- | --- | --- | --- | --- | --- | --- |
| Consumer workflow invoking this action | GitHub API authority | GITHUB_TOKEN input → Octokit | Repository/PR from GITHUB_EVENT_PATH; API reads and COMMENT review write | GitHub API; action runner | GitHub token permissions and event policy supplied by host workflow | src/main.ts:8; src/main.ts:26; src/main.ts:169 |
| Consumer workflow invoking this action | OpenAI request/content export | OPENAI_API_KEY and model inputs; exclude globs filter files | Selected diff chunks plus PR title/body; default model from action input | OpenAI API and action process | Exclude globs; per-call max_tokens 700 limits output, not total requests | action.yml:10; src/main.ts:81; src/main.ts:117; src/main.ts:224 |
| Consumer workflow invoking this action | Posted review payload | Model JSON reviews → body/path/line → createReview | COMMENT event on event-selected repository/PR; path from source diff; line from model | PR readers, GitHub API | No merge/approve operation; JSON.parse plus number conversion, GitHub validates API request | src/main.ts:141; src/main.ts:149; src/main.ts:175; dist/index.js:178 |
| Repository self-review workflow | External action runtime | pull_request opened/synchronize → pinned freeedcom action | Pinned upstream action receives GITHUB_TOKEN/OpenAI secret; write-all declaration | External action runner and API providers | Pinned action/checkout revisions; host event secret restrictions still apply | .github/workflows/code_review.yml:2; .github/workflows/code_review.yml:7; .github/workflows/code_review.yml:15 |

## 2. Threat Model, Trust Boundaries, and Assumptions

A PR author controls title, description and changed code, but does not thereby control runner secrets, the workflow, or repository write permissions. Those attacker-controlled strings enter the same prompt as review instructions, including a system-role message, so generated output must be treated as untrusted even when syntactically JSON. The publication sink grants only the shown review-comment capability; there is no tool call, shell execution, checkout-and-run of PR code, file write, approval or merge operation in this implementation. A poisoned review can mislead humans without granting direct code execution. GitHub controls workflow triggering, fork-secret availability and token capabilities. The maintainer choosing action inputs separately authorizes exporting code to OpenAI; file excludes govern chunks but do not redact title or description. Build/release maintainers can replace the distributable bundle and therefore hold greater authority than ordinary PR authors.

**Assets.** GitHub token and OpenAI key confidentiality; private code when consumers deliberately use the public action on private repositories; review integrity; PR attribution; model/API spending and CI availability. Only this repository’s public source is used to establish these assets.

**Security objectives.** Keep untrusted PR/model text within a bounded review-comment capability. Publish to the intended PR/revision with useful valid locations. Grant minimal GitHub permissions and export only approved content. Bound total work and ensure source and shipped bundle remain consistent.

**Assumptions and unresolved controls.** Runtime code was inspected, not executed. No upstream hosted service claims or current dependency vulnerabilities are inferred offline. The bundled createPrompt/getAIResponse/createReviewComment/main functions support the local TypeScript flow (`dist/index.js:108`, `dist/index.js:178`, `dist/index.js:189`). This repository’s pinned self-review action is external and is not proved identical to the local bundle. README and workflow request write-all, which is broader than COMMENT publication needs, but effective fork restrictions and host token policy require caller context (`README.md:32`, `.github/workflows/code_review.yml:7`). Publication does not explicitly supply a commit_id; a concurrent PR update can affect review location/meaning, but no merge authority is shown. Request consumer workflow permissions, event policy and permitted-data rules when modeling a concrete installation; credentials or private PR contents are unnecessary.

## 3. Attack Surface, Mitigations, and Attacker Stories

These are prioritized hypotheses, not validated vulnerabilities. Priority reflects where review should begin; impact requires the stated prerequisites. The architecture was reviewed independently in a delegated source-only pass before these scenarios were drafted. No application code or external integration was executed.

| Priority | Scenario and capability gain | Prerequisites | Impact | Existing controls | Mitigation | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| P1 | PR author manipulates model into posting misleading review content | Action runs with secrets and valid review-write permission | Bot-attributed misinformation; possible human follow-on action | Sink is COMMENT only; target repository/PR obtained from event; no tools | Treat output as advisory, validate schema/content and keep execution/merge out of model authority | src/main.ts:81; src/main.ts:127; src/main.ts:175 |
| P1 | Runner/action supply chain obtains overbroad token | A separate compromise of shipped action or dependency, plus write-all effective permission | Repository mutation beyond review purpose | Workflow pins action revisions; GitHub permission policy | Use pull-requests write and necessary read permissions; verify packaged artifact provenance | .github/workflows/code_review.yml:7; action.yml:20 |
| P2 | PR content causes large request fanout | Many selected diff chunks or repeated eligible events | API expense, CI delay, review spam | Sequential loop and 700 output tokens per call; excluded files omitted | Cap total chunks/input tokens/comments and cancel superseded work | src/main.ts:65; src/main.ts:117; src/main.ts:224 |
| P2 | Excluded-sensitive information still travels in title/description | Sensitive metadata exists and operator expected excludes to redact all input | Content disclosure to model provider | File-level glob excludes only | Document/export-filter metadata separately; confirm intended data policy | src/main.ts:94; src/main.ts:224 |
| P3 | Model response points at invalid/stale lines | Malformed output or concurrent PR update | Failed publication or misleading review locations | GitHub API validation; COMMENT only | Validate response schema and bind review to analyzed revision | src/main.ts:149; src/main.ts:175 |

## 4. Severity Calibration (Critical, High, Medium, Low)

**Critical.** Critical requires a separate demonstrated path to broad repository or secret compromise. Prompt text alone cannot execute shell commands in this action.

**High.** High may fit a compromised runner using effective broad token grants or unauthorized confidential-code export; a model posting a bad suggestion is normally lower.

**Medium.** Medium fits meaningful bounded review manipulation or spending amplification with actual event reachability; fork PRs without available secrets change feasibility.

**Low.** Low fits invalid comment locations or harmless failed reviews. COMMENT publication is not an approval and cannot be described as automatic merging.

Confidence is limited to inspected source/design evidence. Deployment reachability, real data sensitivity and external authorization must be established separately; uncertainty does not itself raise severity.

Repository: github.com/mathspace/ai-codereviewer

Version: 4de99464040512dffef10b4c3d76288db46d0185
