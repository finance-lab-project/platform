# Architecture and roadmap audit — 2026-10-24

Scope: [platform #15](https://github.com/finance-lab-project/platform/issues/15). This is source and GitHub planning inspection, not a runtime/security/recovery certification. [ADR-031](../architecture/decisions/0031-java-spring-budget-incremental-releases.md) records the maintainer-selected direction; documentation PRs require human review/merge.

## Baseline sources

| Repository / inspected main SHA | Observed state | Reuse / limit |
| --- | --- | --- |
| [Legacy finance-lab `6c7e39758fda11c2a9679feba6cc2d32e14182e3`](https://github.com/anton415/finance-lab/tree/6c7e39758fda11c2a9679feba6cc2d32e14182e3) | React source, Spring T2, ADRs 001–030 and Go Stage 0 planning | Java 21 / Spring Boot 4.1.1 are the existing POM baseline, not a newly selected upgrade. Controller validation/deserialization/tests are reusable; `BudgetService.findByMonth` returns empty and valid PUT returns 501. No database/auth/save implementation found in the scoped backend source/POM. Mocked successful controller reads do not prove real persistence. |
| [Frontend `ece66c213bbdb40095b7d13d5da9d73c7d0abfdd`](https://github.com/finance-lab-project/frontend/tree/ece66c213bbdb40095b7d13d5da9d73c7d0abfdd) | Extracted React/TypeScript/Vite monthly planner; localStorage and JSON v2/CSV tools; README, agent rules, CI and manually approved Pages workflow | Bootstrap #2 and deployment #4 are closed; #5 release publication and #3 dependency remediation are open. The public domain, exact deployed revision and current browser journey were not exercised in this audit. Do not equate closed #4 with a verified v0.1 release. |
| [Go Budget `bc40f19395126ada93c10b76c037b2c752baa5d5`](https://github.com/finance-lab-project/budget-service/tree/bc40f19395126ada93c10b76c037b2c752baa5d5) | Exact Money/month/totals, OpenAPI + fixtures, initial PostgreSQL/Goose schema and `internal/postgres` read adapter | Useful preserved learning/reference work; no executable complete HTTP/auth/save/runtime service. B1 is closed; B2–B4 are unimplemented release plans. Do not overwrite schema or imply Go remains the primary backend. |
| [Platform `6ce94a79f1cea0e012977d3dde77e633d7252e34`](https://github.com/finance-lab-project/platform/tree/6ce94a79f1cea0e012977d3dde77e633d7252e34) | Governance documents and partial Lima/K3s/TLS configuration; freeze #13 and roadmap #14 | #1–11 remain open, including protected CA recovery #8, consumer TLS #9, private ingress #10 and human diagnosis #11. Existing safeguards remain; no VM, secret, certificate, cloud or recovery checks were performed. |
| [Web BFF](https://github.com/finance-lab-project/web-bff) | Empty Git repository; #1–4 are planning/governance issues | No application or root AGENTS implementation to reuse. No bootstrap needed for the current release. |

## Historical decisions and compatibility

- [ADR-030](https://github.com/anton415/finance-lab/blob/6c7e39758fda11c2a9679feba6cc2d32e14182e3/docs/architecture/decisions/0030-release1-go-migration-implementation-sequence.md) directed production work straight to Go/Keycloak/BFF and platform-first delivery, retaining Spring only as migration evidence. ADR-031 explicitly replaces that direction and release sequence, while keeping the original text and history.
- Legacy completed #122/#123 and Go planning/foundation #142–144, PR #153 and organization Go PRs #1/#2/#4/#6/#8 remain completed historical work. The reset does not reopen or relabel them as failed work.
- Legacy T2 #124 expects original string amounts and UUID ownership; #125 expects invited-user Spring sessions. Go v1 instead uses explicit currency, canonical Money, null versus zero, delegated identity and its own revision/replay contracts. A copy/port without reconciliation would silently change behavior. [Platform #16](https://github.com/finance-lab-project/platform/issues/16) owns the next source-backed reuse proposal; neither old design is automatically selected in full.
- Frontend storage/backup semantics and all code/migrations/fixtures remain unchanged. No real personal data or migration is introduced.

## GitHub inventory and disposition

At the pre-change audit, **no open PRs** existed in any of the five inspected repositories (organization search plus complete repository issue/PR listings). Existing closed PRs/history are retained. New PRs from this task are documentation only and await human approval.

| Existing work | Applied planning disposition |
| --- | --- |
| Go Budget #12–18 | Add dated supersession notice and `superseded` label; retain open state, original body and existing mode labels. Optional later standalone learning only; no Release 1 dependency or completion claim. |
| Web BFF #1–4 | Same supersession treatment; no empty-repository bootstrap or mandatory BFF implementation. |
| Legacy frontend #155–157 | Superseded old Go/BFF integration briefs; cross-link the organization frontend #30/#5. Original bodies remain. |
| Legacy Spring #124/#125 | Add `deferred` and a reuse-audit notice. #124 moves from `needs:implementation` to `needs:spec` and from Project Ready to Backlog; #125 stays Backlog. No automatic T2 implementation. |
| Platform #1–11 | Add `deferred` and a dated notice, retaining open state, original evidence, modes and operational/learning gates. Known safety gaps stay unresolved. |
| Platform #13/#14 | Add current decision/roadmap sections, preserving the previous freeze and career plan below. The explicit current request authorizes planning reconciliation only. |
| Frontend #5 | Retain release/tag ownership and human approval; add #30 evidence handoff and explicit frontend-only weekend minimum. No duplicate publication issue. |

No issue is closed as completed or deleted by this reset. No learning issue is closed or assessed. Dated notices distinguish the current direction from earlier acceptance criteria without silently rewriting those criteria.

### Milestones and labels

- Go B2/B3/B4, Web BFF W1 and Platform P1/P2 are prefixed **Deferred**, retain original descriptions after the dated note, and have no delivery deadline.
- Legacy milestone #11 is **Historical — Go Budget migration (superseded by ADR-031)**; #12 is **Superseded — U1 cloud React via Go/BFF**; #9 is **Deferred — T2 Spring reuse (audit before implementation)** with the old November 8 deadline removed. Milestones remain open; their renamed status is not completion evidence. Closed foundation milestones are unchanged.
- Frontend milestone #1 keeps its v0.1.0 identity and targets the end of October 25 in Europe/Moscow. Its only new task is readiness audit #30; #5 remains publication. Frontend v0.2–v0.5 and unrelated legacy milestones remain unchanged.
- `superseded` identifies historical release briefs; `deferred` identifies work outside the current release. These are planning labels, not new lifecycle states or agent dispatch permissions. Existing mode/learning labels remain.

### Projects and governance

The organization has no Projects-v2 board. Existing user-owned Projects are reused: private [Engineering & Career Roadmap #10](https://github.com/users/anton415/projects/10), and public [Finance Lab Development #9](https://github.com/users/anton415/projects/9). Read both projects' fields/items with full pagination; keep private career items/content out of public audit artifacts.

Project #10's product summary/README now identifies React first, Java/Spring next and Go optional. Selected items: frontend #30 and existing #5 Ready, platform #15 Review, platform #16 Backlog. Project #9 retains history and moves legacy #124 to Backlog. Existing statuses, Area/Work type fields, views, workflows, WIP limits and visibility are preserved. The GitHub CLI token has read-only Project scope; these bounded changes use the existing signed-in GitHub UI, without changing permissions.

Repository ruleset lists were empty; main-branch protection reads returned “Branch not protected” for all four nonempty repositories. **Written governance still requires human approval before merge/release/deployment.** No approval bypass, protection change, auto-merge or publication is authorized.

## Follow-ups and approval boundary

- [Platform #16](https://github.com/finance-lab-project/platform/issues/16): bounded Spring reuse/contract/repository-placement audit, then one minimal proposed implementation issue. Not a weekend dependency.
- [Frontend #30](https://github.com/finance-lab-project/frontend/issues/30): exact-revision readiness evidence and synthetic minimum journey. [#5](https://github.com/finance-lab-project/frontend/issues/5): separately approved tag/release publication.
- [Budget #20](https://github.com/finance-lab-project/budget-service/issues/20) and [legacy #162](https://github.com/anton415/finance-lab/issues/162): companion documentation/instruction updates. Main task [platform #15](https://github.com/finance-lab-project/platform/issues/15) links all review PRs.

Documentation verification is limited to diff/scope inspection, historical-text preservation, Markdown path/link checks and GitHub read-back. Application, database, browser, runtime, security and recovery tests were not run because this task changes planning/documentation only. Future audit checklists are planned work, not results. The PR descriptions record the exact observed checks and head revisions.
