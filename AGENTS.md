# AI Repository Context

This repository is the durable configuration and operational record for a small
personal cloud. It is an infrastructure and documentation repository, not an
application-code repository.

The platform should remain understandable, secure by default, observable,
recoverable, inexpensive to operate, and easy to extend. The objective is a
durable engineering platform rather than a collection of running containers.

## Authoritative documentation

Use the following precedence when interpreting the repository:

1. Accepted Architecture Decision Records govern architectural decisions.
2. `HANDBOOK.md` governs stable platform-wide engineering and operational
   standards.
3. `architecture/README.md` describes the current and target architecture.
4. `docs/` contains platform-wide procedures and runbooks.
5. `compose/<service>/README.md` describes service-specific operation.
6. `compose/<service>/docker-compose.yml` is the version-controlled deployment
   definition.
7. `README.md` is the platform overview and current inventory.

The files under `.agents/` summarize this repository for AI tools. They do not
replace the authoritative documents above:

- `.agents/architecture.md`: nodes, trust boundaries, exposure, and storage
- `.agents/services.md`: current service inventory and service-specific facts
- `.agents/workflows.md`: change, validation, onboarding, update, and retirement
- `.agents/known-gaps.md`: accepted limitations and observed repository drift

When documentation and configuration disagree, do not silently choose one.
Identify the discrepancy. Treat the Compose file as evidence of repository
configuration and statements such as "target", "recommended", and "should" as
intended state. Reconcile them only within the scope of the requested change.

## Engineering rules

- Prefer simple, mature, well-understood technology.
- Add complexity only when it removes a larger, concrete constraint.
- Keep changes small, understandable, reversible, and independently verifiable.
- Prefer explicit, declarative configuration over hidden state.
- Treat containers as replaceable workloads.
- Keep configuration, host-specific settings, secrets, persistent state, and
  disposable runtime state separate.
- Treat documentation, monitoring, backup, restore, and recoverability as part
  of implementation.
- A running container is not sufficient proof that an application works.
- Make omissions intentional and document material ones.
- Do not introduce Kubernetes, GitOps, distributed storage, centralized
  secrets, or deeper observability merely as goals.
- Preserve working behavior unless the requested change requires altering it.
- Record significant architectural changes in ADRs rather than allowing
  undocumented implementation drift.

## Repository and data boundaries

Git is the source of truth for Compose definitions, static configuration,
architecture, ADRs, service documentation, runbooks, examples, and reusable
scripts.

Git must not contain runtime databases, uploaded data, generated logs, caches,
model files, backup archives, passwords, tokens, OAuth credentials, private
keys, cookies, recovery codes, tunnel credentials, or local environment files.
Persistent state must be mounted explicitly from outside the repository.

The intended storage convention is:

```text
~/personal-cloud-data/
├── calibre-web/
│   ├── config/
│   └── library/
├── openweb-ui/
│   └── data/
└── uptime-kuma/
    └── data/
```

This is a target convention, not proof that every current Compose file follows
it. See `.agents/known-gaps.md`.

## Sensitive-data handling

- Never display, quote, summarize, or propagate the contents of `.env` files.
- Treat the currently tracked `.env` files as potentially sensitive.
- Do not print fully rendered Compose configuration when it may interpolate
  secrets. Report validation success or relevant sanitized errors instead.
- Use committed `.env.example` files containing variable names and safe
  placeholders; keep real `.env` files local and ignored.
- Do not stage databases, logs, backups, generated configuration, or persistent
  runtime directories.
- Treat credentials stored in application databases as secrets and protect
  their backups accordingly.
- Do not initialize empty application state until an existing mount or data
  location has been verified.
- Inspect `git status`, `git diff`, and `git diff --cached` before committing.

## Change guardrails

- Read the relevant authoritative documents before editing a service.
- Preserve unrelated working-tree changes.
- Do not deploy, restart services, pull images, change Cloudflare or Tailscale,
  or touch live data unless runtime changes are explicitly requested.
- Do not delete persistent data as part of routine container removal.
- Update all affected documentation layers in the same change.
- Validate at the lowest useful level and match validation depth to risk.
- State assumptions that distinguish current state from target state.
- Keep administrative access private.
- Bind same-host tunneled origins to localhost unless broader access is
  intentional and documented.
- Propose an ADR when a change affects a platform-wide pattern, trust boundary,
  storage or deployment model, host role, shared dependency, public ingress, or
  other expensive-to-reverse decision.

