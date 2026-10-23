# Release 1 — organization execution-mode roadmap

**Snapshot:** 2026-10-23. **Product priority:** the first authenticated monthly Budget journey by **2026-12-31**. This map proposes ownership; it does not change release contracts, dependency order, GitHub Project status, `needs:*` labels, service architecture or actual implementation authorization.

See [execution-modes.md](execution-modes.md) and [A1 historical planning index](https://github.com/anton415/finance-lab/issues/146).

## Classification of executable Release 1 work

| Repository and work | Intended mode | Rationale / sequencing |
| --- | --- | --- |
| [budget-service #12 — atomic saves, conflicts and idempotent replay](https://github.com/finance-lab-project/budget-service/issues/12) | `mode:learning` | **First active learning candidate**: human designs transaction/replay strategy and implements a bounded core/test slice after the issue's design checkpoint is approved. Agent tutors/reviews, not auto-codes. No code dispatch by this document. |
| [budget-service #13 — receipt retention](https://github.com/finance-lab-project/budget-service/issues/13) | `mode:delivery` | Bounded follow-on after #12; ensure human understands the retention/recovery boundary. |
| [budget-service #14 — delegated token authorization](https://github.com/finance-lab-project/budget-service/issues/14) | `mode:delivery` | Security-sensitive; human must approve token profile/design and review tests. Can be a later learning exercise **only via explicit mode decision**. |
| [budget-service #15 — Go Budget HTTP read/save API](https://github.com/finance-lab-project/budget-service/issues/15) | `mode:delivery` | Implement verified contract after dependencies. Human checks wire compatibility and security. |
| [budget-service #16 — packaging, health, shutdown](https://github.com/finance-lab-project/budget-service/issues/16) | `mode:delivery` | Repeatable service/runtime work with observed checks and recovery limits. |
| [budget-service #17 — generation fence/restore verification](https://github.com/finance-lab-project/budget-service/issues/17) | `mode:delivery` | Validation with human-observed recovery rehearsal; do not damage real data. |
| [budget-service #18 — end-to-end acceptance tracker](https://github.com/finance-lab-project/budget-service/issues/18) | *No execution label* | Cross-service tracker, not a self-contained implementation assignment. |
| [platform #2 — local Lima/K3s foundation](https://github.com/finance-lab-project/platform/issues/2) | `mode:delivery` | **First delivery pilot**. Existing partial execution remains; platform child issues #8–10 now separately track validation gaps. Continue only after their explicit gates. |
| [platform #8 — protect local CA recovery](https://github.com/finance-lab-project/platform/issues/8) | `mode:delivery` | Human must choose secure destination and explicitly handle private keys; agents cannot upload, inspect or disclose secrets. |
| [platform #9 — real workload TLS and credentials](https://github.com/finance-lab-project/platform/issues/9) | `mode:delivery` | Dependent on actual consumers; no automatic VM restart or Secret modifications. |
| [platform #10 — private ingress verification](https://github.com/finance-lab-project/platform/issues/10) | `mode:delivery` | Preserve no-public-ingress and Mac-only boundaries; separate activation permission. |
| [platform #3 — isolated PostgreSQL](https://github.com/finance-lab-project/platform/issues/3) | `mode:delivery` | Strict credentials, database isolation, recovery evidence. |
| [platform #4 — Keycloak/delegation](https://github.com/finance-lab-project/platform/issues/4) | `mode:delivery` | Human-approved token claims, test identity and rollback boundaries. |
| [platform #5 — GitOps and frontend hosting](https://github.com/finance-lab-project/platform/issues/5) | `mode:delivery` | No unsolicited cloud spending, exposed endpoints or auto-deploy. |
| [platform #6 — private diagnostics](https://github.com/finance-lab-project/platform/issues/6) | `mode:delivery` | Observability implementation. Human analyzes one incident independently. |
| [platform #7 — recovery rehearsal](https://github.com/finance-lab-project/platform/issues/7) | `mode:delivery` | Protect real state; include isolated human recovery evidence. |
| [web-bff #1 — browser/session contract](https://github.com/finance-lab-project/web-bff/issues/1) | `mode:learning` (design only) | Next learning task **after #12**, if selected. Human owns security boundary decisions; no simultaneous learning WIP. |
| [web-bff #2 — login/sessions](https://github.com/finance-lab-project/web-bff/issues/2) | `mode:delivery` | Implement only after the contract checkpoint; carefully gate credentials. |
| [web-bff #3 — authenticated Budget forwarding](https://github.com/finance-lab-project/web-bff/issues/3) | `mode:delivery` | Preserve exact operation identity and no automatic retry. |
| [historical personal-repo frontend #155 — signed-in budget journey](https://github.com/anton415/finance-lab/issues/155) | `mode:delivery` | BFF dependency and real browser integration required. |
| [historical personal-repo frontend #156 — draft/conflict safety](https://github.com/anton415/finance-lab/issues/156) | `mode:delivery` | Preserve draft and pending operation; human acceptance of safe retry semantics. |
| [historical personal-repo frontend #157 — independent SPA release](https://github.com/anton415/finance-lab/issues/157) | `mode:delivery` | Must verify actual deployed browser path; no CI success assumptions. |

The three frontend rows are **historic cross-repository dependencies** until the UI is migrated into an organization repository. Their suggested modes are not applied GitHub labels, nor permission to modify the old repository.

## Supporting and deferred work

- [Platform #11 — Mac mini/Lima diagnostic learning drill](https://github.com/finance-lab-project/platform/issues/11) is `mode:learning`, **Backlog/opt-in** after the foundation stabilizes and after the protected Go learning task. It does not authorize fault injection or a second simultaneous learning WIP.
- [Historical frontend #96 — manual calibration](https://github.com/anton415/finance-lab/issues/96) stays human-led `mode:learning`; the original descriptive `learning` label can remain.
- [Historical AgentOps #154 — synthetic Go Budget adversarial review pilot](https://github.com/anton415/finance-lab/issues/154) is **optional** learning/evaluation, not a Go Budget release prerequisite; proposed `mode:learning` does not authorize mutation or model dispatch.
- [Historical AgentOps #158 — Halfcycle evaluation](https://github.com/anton415/finance-lab/issues/158) is a deferred human-led comparison; no license/purchase or switch from Codex is authorized.
- [A1 tracker #146](https://github.com/anton415/finance-lab/issues/146), [packs #147](https://github.com/anton415/finance-lab/issues/147) and [handoffs #152](https://github.com/anton415/finance-lab/issues/152) remain planning/opt-in. Do not imply the optional personas/automation exist. Preserve one-bounded-pilot limits.
- Historic Spring [#124](https://github.com/anton415/finance-lab/issues/124) and [#125](https://github.com/anton415/finance-lab/issues/125) are not Go migration prerequisites; do not silently implement them.
- Future M5/M6/M7 drafts and feature ideas remain **unclassified until selected**. A descriptive label is not a dispatch signal.

## Organization rollout and implementation gate

The destination is the **`finance-lab-project` organization**, not either legacy personal repository. The installed GitHub integration can create review branches in `platform` and `budget-service`. The BFF repository has no commits yet; no PR/AGENTS.md may exist there until it is initialized through a separately approved initial commit and reviewed instruction PR. Issue labels are **classification only**, with no `needs:*` or Project Status changes.

- Mode labels were applied and verified at creation to 20 organization issues: Budget #12–17, Platform #2–11 and Web BFF #1–4 (including the two new opt-in issues). Budget #18 (integration tracker) and Platform #1 (planning baseline) intentionally remain unlabeled. No old personal-repository issue was relabeled.
- Issue classification reflects **intended mode**. A mode label does not grant an Implementer permission to run tests, change infrastructure, consume paid resources, or create code.
- Review/merge of root `AGENTS.md` PRs and exact human task briefs must precede any new mode-based agent dispatch. The current GitHub tooling has **not** been shown to enforce this automatically.
- The BFF repository is empty, so no root instruction PR exists there yet. [BFF bootstrap tracker #4](https://github.com/finance-lab-project/web-bff/issues/4) and [BFF AGENTS template](web-bff-AGENTS-template.md) preserve the needed work; no direct-to-main first commit was made by this rollout.

## Practical learning acceptance

For the first Go task, the maintainer should: articulate the lock/transaction/replay strategy, implement or independently troubleshoot one accepted nontrivial slice, show observed focused test results, explain a version conflict and an ambiguous commit outcome, and identify at least one design limitation. Reviewer feedback may follow **his own attempt**.

For the Mac mini infrastructure, reuse the existing stand. Plan **one isolated failure/diagnosis rehearsal** after the platform baseline is stable. Require an independently documented hypothesis, observed log/metric evidence, bounded corrective action and verification. Fault injection is not authorized by planning this issue.
