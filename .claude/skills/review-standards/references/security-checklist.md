# Security Review Checklist

Review the diff against these. Most findings here are **blockers** unless clearly outside the system's threat model. Be concrete — name the file, line, and the specific risk.

## Input & validation
- All external input validated at the edge (Pydantic schemas, `Field` constraints). Nothing trusts client/external data raw.
- No SQL built by string concatenation — ORM or parameterized queries only.
- No shell/command construction from user input (`os.system`, `subprocess` with `shell=True` + interpolation).
- File paths from input are validated/sandboxed (no path traversal).

## AuthN / AuthZ
- Every protected route has an auth dependency — check that a new endpoint didn't ship unprotected.
- Authorization (role/ownership) is enforced, not just authentication. A logged-in user shouldn't reach another user's resource.
- Inbound webhooks/callbacks verify a signature or shared secret in constant time, against the raw body.
- Token validation checks signature, `aud`, and `iss` (OIDC) — not just decode.

## Secrets & config
- No hardcoded credentials, API keys, tokens, or connection strings — all via `Settings`/env.
- Secrets, tokens, and full request bodies are not logged.
- `.env`, key files, and dumps are not committed in the diff.

## Data exposure
- Responses expose only intended fields (`response_model`), not whole ORM rows.
- Errors return clean messages — no stack traces, internal paths, or DB errors to clients.
- PII isn't logged or returned beyond what's required.

## Dependencies & misc
- New dependencies are reputable and pinned; flag anything unusual for a quick CVE check.
- CORS isn't wildcard-open on a credentialed API.
- Rate-limiting/abuse considerations noted for new public endpoints (flag if missing, severity by exposure).

## How to report
For each issue: `file:line — risk — exploit scenario in one line → fix`. If you're unsure whether something is in the threat model, raise it as a question rather than asserting a blocker. Never include a working exploit payload — describe the risk and the fix, not a weaponized example.
