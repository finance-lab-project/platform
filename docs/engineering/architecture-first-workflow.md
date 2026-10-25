# Architecture-first AI delivery — Q4 2026

**Owner:** Anton (@anton415). **Setup:** [platform #18](https://github.com/finance-lab-project/platform/issues/18). **Effective proposal:** 2026-10-25; repository changes await independent review and human merge. This document specifies the workflow; it does not demonstrate completed human learning or automated enforcement.

## Outcome and ownership

By **2026-12-31**, demonstrate a bounded Finance Lab MVP, the human's design rationale, and a traceable issue → AI implementation → independent review → CI/security evidence → human decision. Reuse React/TypeScript and the selected Java/Spring direction in [ADR-031](../architecture/decisions/0031-java-spring-budget-incremental-releases.md). PostgreSQL serves an accepted persistence need; Go remains optional product work. Job4j remains active and separately human-owned.

| Responsibility | Owner and evidence |
| --- | --- |
| Problem, scope, acceptance criteria | Human owns and accepts the current issue brief. AI may suggest edits; suggestions are not acceptance. |
| HLD | Human owns system boundary, data flow, storage/identity boundaries, operational assumptions and one meaningful alternative. Reuse the existing HLD/ADR instead of making a document per task. |
| LLD | Human owns the selected increment's contracts, state/data model, failure behavior and test strategy. A short issue section is sufficient. |
| ADR | Human records/accepts consequential choices and tradeoffs. Extend the existing decision when appropriate; routine UI/docs changes may link the current decision and explain why no new ADR is needed. |
| AI Implementer | Implements only the selected brief, tests the change and opens/updates a PR. Does not change acceptance criteria or accept its own design. |
| Independent Reviewer | Separate session/agent or person, not the Implementer's self-check. Reads the brief and actual diff independently, records verdict and findings at the full head SHA; does not fix code during review. A different provider/model is not required. |
| Merge, release, deploy | Human alone decides each action separately against the reviewed revision and actual checks. No auto-merge or autonomous deployment. |

AI drafts do not prove personal design competence. Keep a brief human-authored rationale/demo explanation. Existing protected learning issues remain human + ChatGPT work; Codex does not assess or close them. ShareTrip is presentation-only and later. LFS101/Cisco and career/networking work retain their separate scope; no new courses or parallel product are introduced here.

## One working board

Use the existing private [organization Project #1](https://github.com/orgs/finance-lab-project/projects/1). Its repository Projects may remain for navigation; do not mirror status maintenance manually across several boards. Existing personal Projects #9/#10 are historical context, not another active Finance Lab queue. Private career content stays private.

Preserve **Backlog → Ready → In Progress → Review → Done**, existing Area, Work type, Start date and Target date. Do not add agent/sprint/priority fields or automation merely to imitate the reference Project. Record the sprint window in the issue and use existing date fields. The Board, Backlog, Planning, Roadmap and Completed views already exist.

| Status | Entry/exit rule |
| --- | --- |
| Backlog | Draft, unselected, blocked or deferred. Record the missing dependency/decision in the issue. A calendar target grants no execution permission. |
| Ready | One sprint-sized brief; named human owner; accepted HLD/LLD/ADR links or justified N/A; observable acceptance/checks; dependencies satisfied; exactly one execution mode. Human selects it before dispatch. |
| In Progress | Human has dispatched the accepted brief. One active product implementation at a time. |
| Review | PR/deliverable and actual checks are available; independent review and human decision remain. |
| Done | Acceptance is evidenced and the human accepts the result. Code requires merged PR; a release requires its actual release/deploy evidence. Learning has its own human closure. |

**WIP:** at most 2 In Progress (normally one product and one human learning item), at most 2 Review. Finish/review before starting more. Weekly: inspect the working increment, one human explanation, Job4j progress, blockers and capacity; select only the next bounded brief. When capacity shrinks, reduce scope rather than add parallel agents or missed-deadline pressure.

Keep existing `mode:delivery` / `mode:learning`; at most one existing `needs:*` label when used. No new labels are required. Mode/status labels are not dispatch or merge permission. Future work without a selected mode stays Backlog. Keep deferred infrastructure and preserved Go work there.

## Brief, handoff and review conventions

Use the [issue template](../../.github/ISSUE_TEMPLATE/engineering-task.md), or copy its fields into an **existing** issue in the owning repository. Use the [PR template](../../.github/PULL_REQUEST_TEMPLATE.md) in the same way across repositories; these templates are installed only in platform by this setup.

Freeze a selected brief in a committed Markdown file or other immutable artifact; link its full revision and path in the handoff. Include source/contract SHAs and the human decision reference. A mutable issue or `main` link alone is not a frozen specification. Do not create an extra specification system.

Each PR covers one bounded outcome. Name the actual Implementer and Reviewer separately, include the full tested head SHA and map every acceptance criterion to observed evidence. Use `Closes #N` only for complete issue scope; use `Refs #N` for partial setup. Never close human learning through a product PR.

The Reviewer returns **PASS / CHANGES_REQUESTED / BLOCKED** for the exact head and scope, with severity, file/line and reproduction or a clearly stated evidence gap. No findings means none found within checked scope, not proof of security. Fixes go back to the Implementer. Review the changed surface again after fixes; after two unproductive review rounds, the human resolves the disagreement or narrows scope. Do not add a third adversarial role or mutation campaign by default.

A changed brief/design requires renewed human acceptance. A new code commit invalidates prior head-bound review and merge consent; it does not require re-approving an unchanged implementation brief for an ordinary in-scope fix. No AI-authored comment or account identity counts as human consent.

## Verification and approval gates

| Change | Minimum relevant evidence |
| --- | --- |
| Documentation/process | Diff, relative links, examples/templates, current GitHub metadata and scope. No application tests just to validate prose. |
| Frontend behavior | Existing CI: clean install, lint, tests, build; focused tests and synthetic browser journey for affected behavior. Current baseline commands: `npm ci`, `npm run lint`, `npm test`, `npm run build`. |
| Dependencies / release exposure | Current `npm audit` and `npm audit --omit=dev`, advisory-path/exposure assessment and [frontend #3](https://github.com/finance-lab-project/frontend/issues/3). Green functional tests do not clear advisories. Changes to dependencies need a separate bounded brief. |
| Future Java/PostgreSQL increment | Select checks in [#16](https://github.com/finance-lab-project/platform/issues/16): domain/contract tests and real database integration for claimed persistence. Verify ownership/auth/CSRF before private shared data, and conflicts/retries/durability before claiming those guarantees. No passing claim from stubs or mocked saves. |

Run the smallest checks that establish the changed acceptance criteria, then the owning repo's required baseline. Record exact commands, observed results, CI URLs and full tested SHA. Missing/pending checks are **unknown/pending**, never passed. Use synthetic data; do not publish private prompts, credentials, financial data or local machine paths.

Before merge: final-head independent review, required checks, explicit unresolved-risk disposition, and human code/scope acceptance. Before deploy: a separate human decision for the exact build/artifact, environment, limitations and rollback reference. If the human authored the PR through an agent connection, an authenticated owner signoff is project consent, not a native GitHub self-approval. Never bypass a required GitHub reviewer rule.

### Observed controls and limitations — 2026-10-25

- Main branches of platform, frontend and budget-service reported `protected: false`. Merge/review rules are currently **manual conventions**, not enforced branch protection. No claim that the process prevents a privileged actor from bypassing them; this setup does not change repository administration.
- Frontend CI has verify plus a manually dispatched, main-only deployment job. The `github-pages` environment has required reviewer `anton415`, a main branch policy and `prevent_self_review: false`. The human can release an agent-authored run using their own account; actual human action remains necessary. No deployment was triggered here.
- Platform has no CI workflow in the inspected source; documentation verification is local/manual. Budget-service retains its existing contract/storage workflows for preserved Go work.
- No dedicated security pipeline was observed in these repository trees. Security checks above are brief/PR requirements, not a newly installed scanner or a completed security audit.
- Platform [PR #17](https://github.com/finance-lab-project/platform/pull/17), Budget [PR #21](https://github.com/finance-lab-project/budget-service/pull/21) and the legacy documentation PR remain separate human merge decisions. Unmerged instructions are proposals; selected task briefs must state the applicable approved boundaries explicitly.
- The referenced career conversation exposes user decisions but its assistant plan bodies are unavailable in the retrieved copy. This schedule is a concrete proposal derived from the current request and live issues, not a verbatim recovery of that missing plan.

## Sprint-sized path to December 31

These are **planning targets**, not bulk dispatch or a promise of throughput. Keep Job4j time protected. Select one small implementation outcome per sprint; use existing issues. Do not create all future issues now.

| Window | Bounded outcome | Existing work / exit evidence |
| --- | --- | --- |
| Oct 25–Nov 8 | Establish this workflow; verify the existing frontend release candidate; human accepts backend reuse/design direction | [#18](https://github.com/finance-lab-project/platform/issues/18) and PR #17 for setup; frontend [#30](https://github.com/finance-lab-project/frontend/issues/30) → [#5](https://github.com/finance-lab-project/frontend/issues/5); [#16](https://github.com/finance-lab-project/platform/issues/16) for a bounded audit after the release gate. The Oct 24–25 release target stays separate; if it slips, record the blocker, do not claim success. |
| Nov 9–22 | One useful Java/Spring + PostgreSQL increment selected from the audit, with human HLD/LLD and compatibility decision first | Reuse #16's output; only then create/select one implementation issue in the approved repository. Expected demo: one synthetic monthly budget save/read round trip with real persistence and bounded tests. No repository extraction before the human location decision. |
| Nov 23–Dec 6 | Connect the accepted increment into one end-to-end monthly journey, or finish that same increment if incomplete | Same approved backend brief where possible plus the minimum owning-frontend task. No blanket port of Go contracts; expose private multi-user data only after the relevant security gates. |
| Dec 7–20 | Stabilize the demo and close its highest-impact observed defect | Reuse release/readiness evidence and #3 if dependency exposure is the selected issue. Record failure behavior, synthetic recovery/backup evidence where relevant, limitations and a rollback reference. No new infrastructure milestone. |
| Dec 21–31 | Freeze demonstrable scope; rehearse the human architecture explanation and review evidence; human decides release/publication | Working monthly-budget demonstration, accepted HLD/LLD/ADR links, exact revision, CI/security record, separate Reviewer verdict, and truthful attribution. Reserve this window for evidence and fixes, not new features. |

**MVP evidence:** monthly income/spending/investments and totals, month separation, reload persistence and safe backup/export in the existing frontend; the selected Java/PostgreSQL increment if accepted and delivered; design rationale and the complete governed delivery trail. The backend increment is the target after #16, not an already approved architecture or completed capability. If it cannot safely fit, the human explicitly accepts a reduced frontend-only demo and records the missing backend outcome; do not silently call a stub or reduced scope the intended server-backed MVP.

Configurable categories, extra locales, charts, BFF/Keycloak/K3s/GitOps, new services and generalized orchestration are not mandatory December scope. Optional evaluation in the historical portion of #18 stays deferred. A deadline never substitutes for design, validation or human release approval.
