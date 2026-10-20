# ADR 0023 — `OCP_ALLOWED_ORIGIN_SCHEMES`: admitting browser-extension origins by scheme

- **Status**: Accepted (maintainer sign-off 2026-10-21)
- **Date**: 2026-09-30
- **Amends**: [ADR 0019](0019-inbound-origin-gate.md) (the inbound `Origin` gate), [ADR 0020](0020-declared-hosts.md) (declared hosts)
- **Class**: Hybrid, for the reason ADR 0020 gives — the gate runs before routing, so it sits above Class A, B.1 and B.2 alike. ADR 0006 route (b): a **semantics change** on grandfathered B.2 endpoints, so it carries its own authorization rather than the grandfather clause.

## Context

Browser extensions are OpenAI-compatible clients too: chat sidebars and "use your own endpoint" extensions in Firefox and Chromium POST to `/v1/chat/completions` from the extension's background or popup page. Their requests carry an `Origin`, so ADR 0019's gate applies, and the origin is neither private-range nor same-origin:

```
Origin: moz-extension://<uuid>           (Firefox: a random UUID per install, per profile)
Origin: chrome-extension://abcdefghij…    (Chromium: 32 letters derived from the extension key)
```

The gate answers `403 forbidden_origin` with reason `foreign-origin` — correct under ADR 0019, and the extension's user has no way to tell from the error which setting would admit it.

**What works today, and why it is not enough.** `OCP_ALLOWED_HOSTS` admits one extension: ADR 0020's declared-origin arm compares `parseAuthority(new URL(origin).host)` with the declaration, and for `moz-extension://<uuid>` that host is the UUID, which `parseAuthority` accepts as a DNS-shaped name. Measured: declaring the UUID admits that extension (`declared-origin`) and still refuses any other `moz-extension://` origin (`foreign-origin`). It works by accident rather than by design, and it breaks in the way users actually hit:

- Firefox assigns the UUID **per install and per profile**. Reinstalling the extension, or using it in a second profile, produces a new origin and the 403 returns.
- The declaration is scheme-blind (ADR 0020 §6), so declaring the UUID also admits `http://<uuid>` and `https://<uuid>` — harmless in practice, but not what the operator meant.
- Nothing in the 403, the README or the boot output tells an operator that an extension id belongs in a variable documented as "host names this proxy is served on".

## Decision

Add `OCP_ALLOWED_ORIGIN_SCHEMES` — a comma-separated list of origin **schemes** the operator declares — and admit any origin of a declared scheme.

1. **A new admitting arm in `evaluateOriginGate`**, after the no-origin and GET/HEAD exemptions and before the declared-host, private and same-origin arms. A match is admitted with reason **`declared-scheme`**, distinct from `declared-origin`, so the gate's verdict says which setting admitted a request. The reason is not written to the log on admission: the gate's reason is logged only for a rejection (`origin_rejected`), so the boot output (item 7) is how an operator sees that a scheme is declared.
2. **The comparison is on `new URL(origin).protocol`**, never on the origin string: `https://moz-extension.example` does not match `moz-extension`. An opaque (`null`) or unparseable origin never matches.
3. **`http` and `https` are refused as entries**, reported at boot and dropped. Declaring either would admit every web page and undo ADR 0019 entirely; there is no deployment that needs it, because a web origin is admitted by being private-range, same-origin or declared as a host.
4. **Parsing mirrors ADR 0020's rules where they apply**: split on comma only, entries trimmed, case folded, duplicates collapsed. A trailing `:` or `://` is accepted, so a value pasted from the address bar (`moz-extension://`) works. Anything that is not an RFC 3986 scheme (`ALPHA *( ALPHA / DIGIT / "+" / "-" / "." )`) is reported at boot and dropped — not fatal, for ADR 0020 §7's reason.
5. **`Access-Control-Allow-Origin` echoes a declared-scheme origin**, as ADR 0020 §3 does for declared hosts: without it the gate admits the request and the browser discards the response.
6. **Unset by default.** With the variable empty the gate's behaviour is byte-for-byte unchanged.
7. **Boot prints the admitted schemes**, so an operator can see the widened surface without reading the environment.

`OCP_ALLOWED_HOSTS` remains the way to admit **one** extension and is not changed; this ADR adds the class-wide option beside it.

## Class mapping

- **Class A** — unaffected, for ADR 0020's reason: the gate is reached only when `Origin` is present, and neither `cli.js` nor OCP's outbound call to `api.anthropic.com` sends one.
- **Class B.1** — OpenAI's specification defines request and response fields, not transport-level access control for a self-hosted server. No field is added, removed or retyped.
- **Class B.2** — a **semantics change**: with the variable set, a POST from a declared-scheme origin that previously received `403` now executes, and receives its own origin in `Access-Control-Allow-Origin`. That needs its own authorization (ADR 0006 route (b)), which is this ADR. Not ADR 0012: no field is added. No response key path changes; `scripts/b2-key-snapshot.mjs` reports no difference (see Evidence).

## Consequences

**Trust widens to every extension of the declared scheme.** `moz-extension` admits every extension installed in every Firefox profile of that user, not only the one the operator meant. With the default `ALLOWED_TOOLS` (`Bash`, `Write`, `Edit`, …) any of them can drive tool execution on the host — the exposure ADR 0019 measured, now reachable by installed extension code instead of by web pages. That is an operator opt-in with a smaller population than "the web", on the assumption that the operator knows which extensions that browser runs. The assumption is weaker than it looks: store listings are reviewed by the browser vendors, but sideloaded and enterprise-pushed extensions are not, and a policy-installed extension can be added without the user's action. It is the operator's call, and the README says so next to the variable. An operator who wants one extension declares its id in `OCP_ALLOWED_HOSTS` instead.

**Not affected**: every deployment that does not set the variable; `curl`, the OpenAI SDKs and `ocp-connect` (no `Origin`); every origin already admitted by ADR 0019/0020.

**What this does not do:**

- **Distinguish extensions within a scheme.** Allowlisting individual Chromium extension ids works through `OCP_ALLOWED_HOSTS` today because their ids are stable; Firefox ids are not, which is the gap this ADR closes at the cost of granularity.
- **Change GET handling.** GET and HEAD stay exempt for ADR 0019's reason.
- **Cover non-browser local processes**, which send no `Origin` and were never gated.

## Evidence

- Unit tests in `test-features.mjs`: parsing keeps `moz-extension` / `chrome-extension` and reports `http`, `https://` and a non-scheme; the gate admits any id of a declared scheme with reason `declared-scheme`, refuses the same origin without the declaration (control), and refuses another scheme, a web page, `null`, and a web host whose name contains the scheme. Removing the new gate line turns the admission test red.
- A live-server test (`OCP_ALLOWED_ORIGIN_SCHEMES (live)`, contributed in the #528 review) boots `server.mjs` with `OCP_ALLOWED_ORIGIN_SCHEMES=moz-extension` and requires a `moz-extension://…` `DELETE /cache` to get `200` with its origin echoed in `Access-Control-Allow-Origin`, while a `chrome-extension://…` origin and a web origin on the same instance get `403`. The unit tests cannot see `server.mjs` stop passing the declaration to the gate; this one can. Mutations, each applied to a copy of the tree, Linux (Node 24), `--only OCP_ALLOWED_ORIGIN_SCHEMES` (3 tests at the time, all green unmutated):

  | mutation | red |
  |---|---|
  | drop `declaredSchemes:` from the gate call in `handleRequest` | live test (status `403`) |
  | drop the `matchesDeclaredScheme(...)` arm of the CORS `isAllowedOrigin` line | live test (ACAO not echoed) |
  | parse `undefined` instead of `process.env.OCP_ALLOWED_ORIGIN_SCHEMES` | live test (status `403`) |
  | control: delete the `declared-scheme` arm in `evaluateOriginGate` | live test and the unit gate test |

- Two more live tests, added after an independent review of #528 (a fresh-context Claude Sonnet 5.5 reviewer, not the author of any of this) found two claims above that no test pinned. `OCP_ALLOWED_ORIGIN_SCHEMES (live): declaring a scheme does not widen the CORS echo to any other origin` boots with `moz-extension` declared and requires `GET /health` from a web origin and from a `chrome-extension://…` origin to carry the fallback `Access-Control-Allow-Origin` instead of their own origin (GET is exempt from the gate, so the echo is the only barrier there), with the declared origin's echo on the same path as its control. `OCP_ALLOWED_ORIGIN_SCHEMES (live): unset, a moz-extension origin is refused and not echoed` boots with the variable removed from the child's environment and requires a `moz-extension://…` `DELETE /cache` to get `403` and no echo, with a `DELETE /cache` carrying no `Origin` as its control. Mutations, macOS (Node v26.8.1), `--only OCP_ALLOWED_ORIGIN_SCHEMES` (5 tests, all green unmutated, and still 5 of 5 with the variable exported in the shell):

  | mutation | red |
  |---|---|
  | the CORS line echoes whenever any scheme is declared (`ALLOWED_ORIGIN_SCHEMES.schemes.length > 0` in place of the `matchesDeclaredScheme(...)` arm) | the CORS-echo test only |
  | parse a fixed `"moz-extension"` in place of the environment variable | the unset test only |

  On this tree the earlier rows still hold: dropping `declaredSchemes:` reddens the first live test only; parsing `undefined` reddens it and the CORS-echo test, whose control (the declared origin's echo) fails; deleting the CORS arm reddens the same two.
- Existing ADR 0019 / ADR 0020 tests pass unchanged.
- Live, on Windows: with `OCP_ALLOWED_ORIGIN_SCHEMES=moz-extension`, a POST from an arbitrary `moz-extension://…` origin returned `200` with that origin echoed in `Access-Control-Allow-Origin`; `Origin: https://evil.example` still returned `403`.
- `scripts/b2-key-snapshot.mjs` on Linux (Node 24): "B.2 response key sets match the snapshot (2 profiles)", so no response key path changes.
