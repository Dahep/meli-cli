# meli — CLI specification

**Status:** v1 draft, 2026-11-20. No code yet; this document is the design contract.
**Binary:** `meli`. **Module:** `github.com/Dahep/meli-cli`.
**Companion:** [dotfiles#199](https://github.com/Dahep/dotfiles/issues/199) owns the
integration half — vendored binary, agent skill, per-machine login runbook. The full
primary-source research (2026-11-07, live probes, endpoint status table, terms clauses)
lives in that issue's comments and is the evidence base for every claim below.

## 1. Purpose

A CLI for reading MercadoLibre Chile (MLC) so agents and humans can search live
listings and emit JSON for reports. It is *structurally* read-only: the binary has no
command, flag, or client method that performs a non-GET call against the marketplace
API.

Two consumers:

1. Humans at a terminal — readable tables, prices in CLP.
2. Agents — `--json` output, stable schema, exit codes that say what to do next.

## 2. Non-goals

- **No writes.** No creating/editing/pausing listings, no questions, orders, shipping,
  messages, promotions, moderation, feedback. Not "hidden behind a flag" — absent.
- **No scraping.** No HTML storefront fetching, no bot-wall workarounds. The developer
  terms prohibit it (§7.6) and the storefront blocks blocked clients anyway.
- **No multi-site.** The site is fixed to `MLC`. Overrides (`--site MLA`, `MELI_SITE`)
  are refused with exit 2, not silently ignored.
- **No seller-private data.** No `/users/{id}/items/search`, no orders/questions reads.
- **No cache, no service.** Every command hits the API live. The terms require item
  lists be at most 6 h stale (§5.3); a zero-cache CLI satisfies that by construction.
- **Not an MCP server.** If that ever exists it wraps this binary; it does not replace
  the read-only guarantee, which lives in the client package.

## 3. Read-only posture (enforced, testable)

Three independent layers, from strongest to weakest:

1. **No write surface in the binary.** The data-plane client (`internal/meli`) exposes
   exactly one method, `Get(ctx, path, query)`. There is no generic
   `request(method, ...)` exported or reachable from it. Enforced by
   `TestClientIsGETOnly`, which AST-scans `internal/meli` and fails on any
   `http.NewRequest*` whose method argument is not the constant `http.MethodGet`.
2. **Meli-side scope.** The registered application is read-scope; Meli rejects any
   mutation from such an app with an authorization error regardless of caller.
3. **Command set.** `TestHelpCommandSet` asserts `meli --help` lists exactly the
   commands in §5 — a new command cannot land without updating the test, so a write
   verb cannot be added quietly.

The one non-GET request in the codebase is `POST /oauth/token` in `internal/auth`,
which exchanges and refreshes tokens. It lives outside the data-plane packages, cannot
be reached through `internal/meli`, and takes no caller-supplied URL. A second AST test
(`TestOnlyAuthPostsToOAuth`) pins that: POST exists in `internal/auth` only, and its
path is the literal `/oauth/token`.

## 4. Authentication

### 4.1 Verified facts (do not re-derive)

From Meli's own documentation (sources in the research note):

| Fact | Value |
|---|---|
| Access token lifetime | 6 h (`expires_in: 10800`) |
| Refresh token lifetime | 6 months, **single-use**; only the newest is valid |
| Idle invalidation | 4 months without an API call |
| Early invalidation | password change, app secret rotation, user revocation |
| Token endpoint | `POST https://api.mercadolibre.com/oauth/token`, form-encoded **body**, JSON response |
| Documented errors | `invalid_grant`, `invalid_client`, `unsupported_grant_type`, `local_rate_limited` (429) |
| App creation (Chile) | may require KYC-validated account-holder details — lead-time risk |
| Scopes | read = GET; a read-scope app's writes error at Meli |

### 4.2 Grant

**`authorization_code` with a paste-code flow** is the v1 path, because it needs no
device-grant guesswork and no end-user secret handling:

1. `meli login` prints an authorization URL for
   `https://auth.mercadolibre.com.cl/authorization?response_type=code&client_id=…&redirect_uri=…`.
2. The user opens it anywhere (desktop browser, or phone when the box is headless),
   authorizes their own account, and copies back the `code=` parameter — the bare code
   or the whole redirect URL.
3. The CLI exchanges it at `/oauth/token` and persists access + refresh tokens.

`client_credentials` is **UNVERIFIED against search** (the endpoint accepts the grant;
whether the resulting token authorizes `/sites/MLC/search` is unknown without a real
app). It therefore ships only behind an explicit `meli login --client-credentials`
attempt, whose outcome is recorded in the runbook; the documented paste-code path is
the default and the one agents are told about.

### 4.3 Where the token lives

Login picks exactly one store and says which, on stdout, every time:

- **Keyring** (preferred): Secret Service via `secret-tool`
  (`lookup`/`store` under service `meli`, account `oauth`).
- **File** (headless fallback): `${XDG_CONFIG_HOME:-~/.config}/meli/credentials.json`,
  created `0600` inside a `0700` directory, written atomically (tempfile + rename).

Rules:

- Never anywhere else: not in the repo, not in a dotfiles-tracked path, not in shell
  profiles, not in logs. `--verbose` output is scrubbed of token material (tested).
- A file store with looser permissions is refused with instructions to `chmod 600`,
  not silently used.
- The app's `client_secret` is a credential and lives in the same store as the tokens;
  only `client_id` and `redirect_uri` are non-secret config (§9).

### 4.4 Refresh discipline

The single-use refresh token is the sharp edge:

- Refresh proactively when the access token is within **5 minutes** of expiry, and
  reactively **once** on `401 {"code":"unauthorized"}`.
- Hold an exclusive lock (`flock`) across read → refresh → write so concurrent CLI
  invocations cannot invalidate each other's token; the waiter re-reads after the lock.
- **Persist the new refresh token before using the new access token.** A crash between
  the two must not lose the only valid refresh token.
- `invalid_grant` ("expired or already used") is fatal on arrival: no retry loop, no
  second attempt. Exit 3 with `run: meli login`.

## 5. Commands

| Command | Purpose |
|---|---|
| `meli login [--client-credentials]` | Paste-code flow; persist tokens; print store kind/path |
| `meli logout` | Delete tokens (and client_secret if stored) from the store |
| `meli status` | Store kind + path, token age, expiry window, idle age |
| `meli whoami` | `status` plus `GET /users/me` identity (id, nickname, site) |
| `meli search` | Search listings (the primary command) |
| `meli item <id>` | One listing, optionally with description |
| `meli categories [id]` | Category tree (site root) or one category node |
| `meli trends [category]` | Trending search keywords for MLC |
| `meli version` | Version, commit, build date |

`meli search` flags: `-q/--query`, `--category`, `--price-min`, `--price-max`,
`--limit` (default 20, max 50), `--offset`, `--sort`
(`relevance|price_asc|price_desc|newest`), `--json`.

Every command accepts `--json`, `--timeout` (default 15s), and `--verbose` (stderr
diagnostics, no secrets). `meli help <cmd>` / `--help` are the agent-facing docs and
are written for that: what the command does, one example invocation, the exit codes it
can produce.

### Exit codes

| Code | Meaning | What the caller should do |
|---|---|---|
| 0 | Success | — |
| 2 | Usage error (bad flag, refused site override, bad id shape) | Fix the invocation |
| 3 | Auth required / expired / revoked | `meli login` |
| 4 | Rate limited after retries | Wait; the message carries the retry hint |
| 5 | Not found (item id, category id) | Check the id |
| 1 | Anything else (network, unexpected body, `PolicyAgent` block) | Read stderr |

## 6. Output contract

`--json` is the canonical machine output: **one JSON object on stdout**, nothing else
on stdout. Human text, progress, warnings, and errors go to stderr only.

### Search

```json
{
  "site": "MLC",
  "query": "mate",
  "fetched_at": "2026-11-20T15:04:05Z",
  "paging": { "limit": 20, "offset": 0, "total": 1327 },
  "results": [
    {
      "id": "MLC1234567890",
      "title": "Mate cerámico …",
      "price": 12990,
      "currency_id": "CLP",
      "condition": "new",
      "permalink": "https://articulo.mercadolibre.cl/MLC-1234567890-...",
      "thumbnail": "https://http2.mlstatic.com/...",
      "available_quantity": 12,
      "sold_quantity": 34,
      "shipping": { "free_shipping": true },
      "seller": { "id": 123456, "nickname": "tienda", "reputation": "green" }
    }
  ]
}
```

Rules:

- Numbers are numbers (CLP has 0 decimals; no floats for money in the JSON — `price`
  is an integer when the API returns one, otherwise a decimal string is preserved).
- Unknown fields are **omitted**, never `null`, so consumers can test presence.
- `fetched_at` is RFC3339 UTC; struct marshalling gives stable key order per type.
- Trimming is deliberate: agents render tables, so the field list above is the
  contract. Adding a field is a minor version bump; renaming one is major.

Human output without `--json`: aligned table — price, title, condition, seller
reputation, id — truncated to terminal width, plus a one-line summary
(`20 of 1327 results · MLC · CLP`).

## 7. Resilience

- **Rate limits:** Meli publishes no numbers; control is per Client ID per endpoint.
  The client is sequential (no concurrent requests), self-limits to 1 request/second,
  honors `Retry-After` when present, and retries 429/5xx up to 3 times with
  exponential backoff plus jitter before exiting 4.
- **Branch on the body, not the status.** The same status means different things:
  `{"message":"forbidden","error":"forbidden"}` on `/search` = no/garbage token (exit
  3); `{"code":"unauthorized","message":"invalid access token"}` = expired (refresh
  once, then exit 3); `{"blocked_by":"PolicyAgent"}` = endpoint policy block (exit 1
  with a hint); `/orders`-style `not_identified_user` = exit 3.
- **Response shapes are documented per endpoint and parsed strictly** into trimmed
  structs; an unexpected shape is exit 1 with the endpoint name, never a zero-value
  silently rendered.
- Timeouts everywhere (default 15s per request, `--timeout` overrides). No unbounded
  reads.

## 8. Architecture

```
cmd/meli/            entry point: flag parsing, command dispatch, exit-code mapping
internal/meli/       data plane: GET-only client + one file per resource
internal/auth/       the only POST: /oauth/token exchange, refresh, rotation
internal/credstore/  keyring (secret-tool) and 0600 file stores + flock
internal/output/     JSON encoder, table renderer, CLP formatting
```

- **Zero third-party dependencies** (`go.mod` has no `require` block). The dependency
  surface is part of the read-only guarantee: nothing in the tree can originate a
  request the client package did not authorize. Command dispatch is hand-rolled;
  `--help` quality is an acceptance criterion, not a reason to add a framework.
- `internal/meli` never imports `internal/auth`; the entry point wires a token source
  into the client as a function. This keeps the POST physically out of the data plane
  and makes the AST tests meaningful.
- Tests use `httptest.Server` against the client's configurable base URL
  (`MELI_API_BASE`, test-only); no test talks to the real API except the opt-in
  `//go:build live` suite.

## 9. Configuration

- `${XDG_CONFIG_HOME:-~/.config}/meli/config.json` — `client_id`, `redirect_uri`,
  optional `default_limit`. Non-secret only; `0600` anyway for simplicity.
- Credentials (access token, refresh token, `client_secret`) — §4.3 store, never in
  this file.
- Env overrides for scripting and tests: `MELI_CLIENT_ID`, `MELI_CLIENT_SECRET`
  (bypass the store; documented as "CI only"), `MELI_CONFIG_DIR`, `MELI_API_BASE`
  (test-only). **`MELI_SITE` is refused**, not honored.
- First-run `meli login` with no `client_id` prompts for it and writes the config.

## 10. Acceptance

A release is shippable when all of these hold:

- `gofmt -l .` empty, `go vet ./...` clean, `go test ./...` green on linux/amd64, and
  `go.mod` has zero `require` lines.
- `TestClientIsGETOnly` and `TestOnlyAuthPostsToOAuth` pass (§3).
- `TestHelpCommandSet` passes; `meli --help` and each `meli <cmd> --help` read cleanly
  with no reference to any write capability.
- `TestNoTokenInOutput`: `meli search --json --verbose` output contains no substring of
  the access or refresh token.
- `TestTableOutput` golden file; `TestSearchJSON` golden file against a recorded
  response.
- Logged out: `meli status` exits 3 with `run: meli login`. Expired token: one refresh
  attempt, then exit 3. Reused refresh token: exit 3, no retry.
- On a logged-in machine: `meli search --json -q mate --limit 3` exits 0 and yields at
  least one real MLC listing with a `permalink`.
- `meli search --site MLA` exits 2.
- Release artifacts: `meli-vX.Y.Z-linux-amd64` (+ `linux-arm64` when built) and a
  `meli-vX.Y.Z-sha256.txt`; both attached to the GitHub release.

## 11. Distribution

- Tags `vX.Y.Z` on `main`; `go build` cross-compiled per platform (goreleaser optional,
  a small `Makefile` is enough).
- sha256 sums are mandatory — the dotfiles repo vendors this binary pinned by version
  and hash (`scripts/setup-meli.sh`, the `setup-vultr-cli.sh` pattern).
- The binary is statically linked (`CGO_ENABLED=0`) so one artifact per platform works
  on both omarchy and hersan.

## 12. Open questions

1. **`client_credentials` authorization:** does such a token actually authorize
   `/sites/MLC/search`? Unverified — first ticket, needs real app credentials.
2. **App provisioning:** Chilean app creation may require KYC-validated account-holder
   details. Human step with lead time; blocks nothing else.
3. **Keyring + concurrency:** whether `secret-tool` alone can serialize refresh
   adequately, or the lock must live in a sidecar file regardless of store.
4. **Category listing gate:** `/sites/MLC/categories` returned 403 anonymously while
   `/categories/{id}` returned 200. Re-probe after login; keep `meli categories`
   dependent on a valid token either way.
5. **Numeric rate limits:** unpublished. The 1 rps default is conservative guesswork
   until real 429s appear.

## 13. First tickets

1. Verify the grant and record it: create the app, run both flows, write the result
   into the runbook (human + code).
2. Skeleton: module, zero-dep dispatch, `version`, `internal/credstore`, `internal/auth`,
   `login`/`status`/`logout` — with the AST tests from §3 landing in the same PR.
3. Data plane: GET-only client, `search` with the §6 JSON contract and table output.
4. `item`, `categories`, `trends`.
5. Release pipeline + sha256; hand the pinned version to dotfiles#199's
   `setup-meli.sh`.

## 14. Legal notes that shaped the design

- No scraping (§7.6); API only.
- Item lists must be ≤ 6 h old (§5.3); no caching is the simplest compliance.
- No redistribution outside the application and no derived aggregate statistics without
  written authorization (§7.2, §7.4, §5.4) — the CLI prints raw listing fields only; it
  computes no aggregates.
