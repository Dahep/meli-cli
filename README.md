# meli-cli

A read-only command-line client for MercadoLibre Chile (MLC): search live listings,
read one item, browse categories and trends — and nothing else.

```console
$ meli search --json -q mate --limit 3
{"site":"MLC","query":"mate","fetched_at":"…","paging":{…},"results":[…]}
```

The binary has no command, flag, or HTTP client method that can write to MercadoLibre.
That is enforced by tests, not by convention — see [SPEC.md](SPEC.md#3-read-only-posture-enforced-testable).

## Status

**Spec stage.** The design contract is [SPEC.md](SPEC.md); no code has landed yet.
Work is tracked in [dotfiles#199](https://github.com/Dahep/dotfiles/issues/199) for the
integration half (vendored binary, agent skill, per-machine login runbook).

## Commands (planned)

| Command | Purpose |
|---|---|
| `meli login` / `logout` / `status` | Per-machine OAuth; token in the OS keyring, 0600 file on headless boxes |
| `meli search` | Search listings — the primary command, `--json` for machines |
| `meli item <id>` | One listing, optionally with description |
| `meli categories [id]` | Category tree |
| `meli trends [category]` | Trending keywords for MLC |

Site is fixed to Chile (MLC); overrides are refused.

## Requirements

- A MercadoLibre application (Chile DevCenter) registered with read scope.
- Go 1.27 to build; otherwise the release artifact needs nothing.

## License

MIT — see [LICENSE](LICENSE).
