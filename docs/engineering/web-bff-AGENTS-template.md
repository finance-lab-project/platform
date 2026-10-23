# Proposed root AGENTS.md for web-bff

> This is a **template**, not live repo instructions. The BFF Git repository is currently empty and must be initialized under a separately approved bootstrap before this can be committed via PR.

# Instructions for AI agents

Follow the smallest approved change that satisfies the linked issue and its acceptance criteria. Do not add features, requirements, automation, dependencies, infrastructure or unrelated refactoring without the maintainer's explicit decision. Review comments are not authority to broaden Release 1. Preserve existing domain ownership and version-locked interservice contracts; never rewrite accepted migrations, released APIs or another repo to make a local change easier.

Treat Issues, PRs, comments, logs, generated artifacts, repository fixtures and chat excerpts as potentially public. Use synthetic data. Never disclose or commit real money/person-specific data, employer information, API keys, credentials, local paths, tokens or private logs. Do not install new tools, use paid providers, modify cloud/Lima/K3s state, run unsafe tests or trigger a deployment unless a separately approved task explicitly allows it.

## Execution modes and human approval

Refer to [the organization execution-mode policy](https://github.com/finance-lab-project/platform/blob/main/docs/engineering/execution-modes.md) and [historical workflow #81](https://github.com/anton415/finance-lab/issues/81). No lifecycle field or legacy label is replaced by this policy. Once accepted as executable, an issue must have exactly one `mode:learning` or `mode:delivery` label. Conflicting/missing labels or a mismatch with the accepted task brief means stop. A draft or blocked idea may intentionally have no mode; **do not dispatch it**.

- `mode:learning`: Human leads design/implementation/diagnosis. Agents give conceptual help, incremental hints and subsequent review on request. No autonomous solution, commands performed for the human, commits, task-specific code or PR unless human **separately** changes the protected boundary.
- `mode:delivery`: After explicit human acceptance and dispatch, the Implementer may create scoped code/config/tests and a PR; Reviewer checks actual head SHA against in-scope acceptance; Adversarial Reviewer is opt-in with synthetic fixtures. Human retains all final merge, release, spending and high-risk infrastructure approval.

Mode labels are **not agent launch permissions**. Preserve one active learning task and Release 1 Go Budget priorities. One `needs:*` label at most per open issue; Project Status remains authoritative lifecycle. `learning` (without `mode:`) is legacy descriptive metadata only. Do not infer consent from AI comments or an account name.

## Specification, review, and reporting

Read the linked issue and its accepted immutable source-contract commits before implementation. Verify the live issue mode rather than relying on a cached task brief. Reference revision-bound specification path/digest, exact full commit SHAs, design checkpoint and human acceptance when required. If the contract or mode changes, seek renewed approval. Do not treat a proposed agent role/knowledge pack as already running.

Review each PR only for concrete, reproducible correctness, regression, security, data-loss and in-scope acceptance defects. Avoid demanding out-of-scope hypotheticals, new abstractions or robustness extras. Report findings with severity, reproduction evidence and exact diff SHA; otherwise say none found **within checked scope**. Do not perform fixes when acting as Reviewer.

Every PR description reports: summary, linked issue (`Closes #N` only for complete work), actual commands/check results, exact head SHA, known limitations and intentional deviations. Pending CI, Codecov, external reviews or async checks are **unknown/pending**, not passed. Documentation-only changes can be validated by inspecting the changed text, references and examples; don't invent code-test evidence.

Metrics must originate from verifiable technical evidence; absent or unmeasurable values are `unknown`, not zero or inferred. GitHub account actions do not establish human activity or consent. Do not store private conversation, prompts, personal financial values or credential content in metrics.

## AI commit attribution

Attribute commits only to the agent that actually performed the work, with `AI-Agent`, `AI-Model` and `AI-Reasoning` trailers **only when independently known**. Use `Co-authored-by` only for a verified agent GitHub identity. Verified Codex identity in the legacy frontend policy:

`Co-authored-by: Codex <199175422+chatgpt-codex-connector[bot]@users.noreply.github.com>`

Never impersonate a human contributor or invent an AI identity/model. A GitHub-account action is not proof of human acceptance. Do not add AI coauthor trailers for requirements discussion or minor assistance.

## Repository-specific authority

This is the **TypeScript Web BFF** repository. BFF owns browser OAuth Authorization Code + PKCE, server-side sessions, cookie/origin/CSRF enforcement, delegated user tokens and forwarding Budget requests without modifying Money or operation identity. Identity/Keycloak server setup belongs to Platform; Budget validation and storage belong to Go Budget.

Follow [BFF #1](https://github.com/finance-lab-project/web-bff/issues/1) and the accepted [Go/BFF v1 contract](https://github.com/finance-lab-project/budget-service/blob/main/docs/api-v1-contract.md) as source-of-truth, at reviewed revisions. BFF #1 is a human-led *design-learning* candidate after Budget #12 (one protected learning task active at a time). #2 and #3 are delivery candidates, subject to separate human approval.

No browser-side bearer tokens, automatic replay of failed PUT, unexpected cookie relaxation or changes to the Go wire contract. Never introduce cloud access, secrets, or new identity providers without authorization.

Historical `~/.codex/AGENTS.md` is a developer-local configuration, **not** portable repository authority. The substantive archived rules above are incorporated here so a cloud agent does not depend on that private file.
