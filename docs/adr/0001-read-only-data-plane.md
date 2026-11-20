# Read-only by construction, not by convention

`meli` must be unable to mutate a Meli account, and "we ship no write commands" does not make that true — a general-purpose HTTP client invites the next contributor to add one. So the data plane exposes exactly one retrieval method and no caller-supplied request shape, the single non-retrieval call in the tree is the token exchange in its own package, and two AST tests pin both facts. Considered and rejected: a generic `request(method, path)` client (one caller away from a write), and relying only on the marketplace's read-scoped application (that is a server-side setting this repo does not control).

Status: accepted. Contract: `SPEC.md` §3.
