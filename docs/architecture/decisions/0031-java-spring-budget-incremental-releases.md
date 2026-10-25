# ADR-031: Java/Spring Budget and small incremental releases

- **Recorded:** 2026-10-24
- **Decision:** Direction explicitly approved by the maintainer; this record awaits human PR review/merge. It does not approve implementation, deployment or publication.
- **Supersedes:** [ADR-030](https://github.com/anton415/finance-lab/blob/6c7e39758fda11c2a9679feba6cc2d32e14182e3/docs/architecture/decisions/0030-release1-go-migration-implementation-sequence.md) for primary Budget technology and the Release 1 delivery sequence.
- **Owner:** [platform #15](https://github.com/finance-lab-project/platform/issues/15)

## Context

The previous Spring T2 → Go plan made a first usable budget depend on a Go service, BFF, identity platform and substantial infrastructure. The [2026-10-24 audit](../../engineering/architecture-reset-audit-2026-10-24.md) found a working frontend source baseline, reusable Spring validation/controller scaffolding, and useful but incomplete Go domain/contract/storage work. Building the entire former architecture would delay the smallest usable product.

## Decision

1. **Java + Spring Boot is the primary Finance Lab Budget backend.** Reuse the existing T2 work where it fits the next accepted increment. Do not complete Spring merely as a temporary step toward Go. Repository placement/extraction remains a decision in the [Spring reuse audit](https://github.com/finance-lab-project/platform/issues/16); this ADR creates no new service/repository and does not overwrite the Go repository.
2. **React + TypeScript remains the frontend. PostgreSQL is the simple database target when server persistence is needed.** Prefer a small application with one Budget backend and its database; do not require a separate BFF, Keycloak, K3s, GitOps or distributed messaging merely to implement Budget.
3. **First priority: a tiny functional version during October 24–25, 2026.** A frontend-only static release with browser storage is valid. Reuse the current monthly income/spending/investments, totals, month isolation and backup flow. Verify the real user journey and limitations under [frontend #30](https://github.com/finance-lab-project/frontend/issues/30); [frontend #5](https://github.com/finance-lab-project/frontend/issues/5) owns the human-approved tag/publication. The date is a target, not evidence of completion or permission to bypass review.
4. **Add backend/database capability incrementally after a concrete need and bounded issue are accepted.** Audit T2 first, then choose a small Spring/PostgreSQL increment with observed tests. Reassess the old T2 persistence/session briefs; do not execute their full design automatically. Authentication/authorization, user/month isolation and CSRF are required before exposing private multi-user data; durable saves, concurrency and retry handling must be verified before claiming those capabilities. Frontend-only scope has no server capability claim.
5. **Go is a later secondary standalone service or CLI learning option.** Preserve all Go source, migrations, tests and contracts. The old Go Budget production/migration briefs are superseded for Release 1 and are not delivered. A future Go exercise needs its own small goal, independent value, acceptance criteria and explicit human selection. It is never a hidden prerequisite for the Java backend or first release. Preserve protected human learning ownership.
6. **Do not silently port contracts or data.** Spring/V2 preserves amount strings and blank values; Go v1 specifies canonical decimal Money, explicit currency and distinct null/zero. These are different contracts. Keep current browser storage, backups, fixtures and applied migrations unchanged. Audit compatibility before any separately approved conversion/API change; do not infer currency or claim old data migration is necessary.
7. **Reduce the release critical path.** The former multi-service requirements in ADRs 001–029 remain historical design references wherever they conflict with this smaller direction. They do not impose Portfolio, FIRE, Kafka, Analytics, Redis, MongoDB, product AI, Keycloak/BFF or Kubernetes on this release. Existing safety, data integrity, human approval and cost constraints are not relaxed. The 2026-10-25 plan targets a demonstrable bounded MVP by December 31 using the [architecture-first workflow](../../engineering/architecture-first-workflow.md); it is not a promise to deliver the former stack. Backend scope and repository placement still require the #16 human decision.

## Delivery order

| Order | Outcome | Gate |
| --- | --- | --- |
| Now: Oct 24–25 | Verify and publish the smallest working frontend slice | Frontend #30 evidence → #5 human release decision; existing #3 dependency triage remains visible |
| Next | Audit Spring T2 reuse and choose the smallest useful Java/PostgreSQL increment | Platform #16; reviewed repository/contract decision and a separate implementation issue |
| Later | Add only selected server-backed capabilities and UI improvements | Measurable need, bounded acceptance, tests and human approval; UI versions v0.2–v0.5 remain optional sequencing, not backend prerequisites |
| Optional later | One standalone Go service/CLI learning exercise | Separate human-selected scope, independent of Release 1 |

## Consequences and retained history

- Existing frontend limitations remain explicit: experimental local storage, no account, synchronization or server backup. Nothing here certifies a public deployment or release.
- Spring GET currently uses a stub and valid PUT still returns 501; reuse is not a completed backend. Go does not become production-complete by preserving it.
- ADR-030, the Go Stage 0 specification and completed issues/PRs remain linked historical evidence. Superseded/deferred issues retain their original bodies and are not marked completed.
- No runtime, schema, API, secrets, infrastructure, code migration or repository ownership changes are authorized by this documentation task. All resulting PRs await explicit human approval before merge.

## Follow-up and sources

See the [current organization roadmap](../../engineering/current-roadmap.md), [audit/disposition record](../../engineering/architecture-reset-audit-2026-10-24.md), [freeze #13](https://github.com/finance-lab-project/platform/issues/13), and [roadmap index #14](https://github.com/finance-lab-project/platform/issues/14). Old planning text is evidence of earlier decisions, not current dispatch authority.
