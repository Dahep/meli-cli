# No cache, ever

The developer terms require that item lists shown to anyone be at most six hours old, and a cache is the one component that turns that obligation into ongoing operational work. Every command therefore hits the marketplace live, which satisfies the staleness clause by construction and removes a whole class of "is this number stale?" bugs from reports. Considered and rejected: a short-TTL on-disk cache (reintroduces staleness computation, invalidation and a second store to secure), and an in-memory cache within one process (buys nothing at the call volumes this CLI sees).

Status: accepted. Contract: `SPEC.md` §2, §14.
