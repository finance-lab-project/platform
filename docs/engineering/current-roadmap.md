# Finance Lab current product roadmap

Updated 2026-10-25 under [ADR-031](../architecture/decisions/0031-java-spring-budget-incremental-releases.md). Primary backend: **Java/Spring Boot**. Frontend: **React/TypeScript**. Database when needed: **PostgreSQL**. Go is optional later standalone learning, outside Release 1.

| Priority | Canonical work | Status / boundary |
| --- | --- | --- |
| Weekend: Oct 24–25 | [Frontend #30: readiness evidence](https://github.com/finance-lab-project/frontend/issues/30), then [#5: tag/release](https://github.com/finance-lab-project/frontend/issues/5) | Tiny frontend-only version is sufficient if functional. #2 bootstrap and #4 deployment are closed; actual release evidence still needs verification. No backend dependency. |
| Release risk follow-up | [Frontend #3: development/test dependency advisories](https://github.com/finance-lab-project/frontend/issues/3) | Retain scoped static-build exposure assessment; do not infer security from green tests. |
| Next backend decision | [Platform #16: Spring T2 reuse audit](https://github.com/finance-lab-project/platform/issues/16) | Source/contract audit and one small proposal only. No implementation or repository move. |
| Later UI | [Frontend milestones](https://github.com/finance-lab-project/frontend/milestones) v0.2–v0.5 | Optional sequence, no extra weekend scope or guaranteed dates; not prerequisites for adding a backend. |
| Historical migration | Go #12–18, BFF #1–4, legacy UI #155–157 | Superseded as Release 1 briefs. Original bodies/history remain; revisit only through a separately accepted issue. |
| Deferred infrastructure | Platform #1–11 | Frozen/opt-in, outside the release dependency chain. Known recovery/TLS gaps remain open; deferral does not establish safety or authorize operation. |

The [audit](architecture-reset-audit-2026-10-24.md) records baseline revisions, limitations and exact issue/milestone dispositions. [Platform #15](https://github.com/finance-lab-project/platform/issues/15) tracks this planning change and its review PRs.

Use the existing private [organization Engineering & Career Roadmap](https://github.com/orgs/finance-lab-project/projects/1) as the current Finance Lab queue. Preserve the personal Projects #9/#10 as historical context and keep private career content private. Do not duplicate all old backlog items or move private content into a public board. Status and labels are not execution permission. Keep human learning separate from software acceptance.

The [architecture-first workflow](architecture-first-workflow.md) defines human design ownership, independent review, issue/PR conventions, existing enforcement gaps and sprint targets to a demonstrable MVP by **2026-12-31**. Its dated update supersedes the old progress-checkpoint-only deadline, without restoring the former distributed stack. Reuse [#18](https://github.com/finance-lab-project/platform/issues/18) for this setup; the optional historical experiment remains deferred.

The [old Release 1 execution map](release1-execution-map.md) is historical. No agent dispatch, implementation, publication, merge, infrastructure change or spending follows automatically from this roadmap.
