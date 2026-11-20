# AGENTS.md

meli-cli — `meli`, a read-only MercadoLibre Chile (MLC) CLI: search live listings and emit JSON for agents and humans. `SPEC.md` is the design contract; domain language lives in `CONTEXT.md`; design decisions in `docs/adr/`.

## Shipping

Work lands as a branch plus a PR; never commit to or push `main` directly, and never rewrite its history — the machine-wide rules in `~/.config/opencode/AGENTS.md` apply here unchanged. The initial setup commit (`0d2000d`, the engineering-skills scaffold) was a one-time exception granted explicitly for that setup and is **not** a precedent: the repo has no merge gate yet, but that is not permission to bypass review.

## Agent skills

### Issue tracker

GitHub Issues on `Dahep/meli-cli`, accessed with the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical triage roles, each label string equal to its role name. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: root `CONTEXT.md` + `docs/adr/`. Per the `domain-modeling` skill.

### Model pins

Subagent dispatch must name an explicitly pinned seat — a bare `task` call silently inherits the parent's model. The dispatch roster, billing split, and per-seat pins live in the machine-wide agent rules (`~/.config/opencode/AGENTS.md`, stowed from the dotfiles repo); pinned seat defs live in `~/.config/opencode/agent/*.md`. This is a personal-side project: dispatch the bare seat names, never the `w-` variants. Pin changes belong to the `model-rotation` skill, never to ad-hoc edits. Provider billing is constrained by the stamp in `.opencode/opencode.json` (`--personal`): only the personal keys can bill here.
