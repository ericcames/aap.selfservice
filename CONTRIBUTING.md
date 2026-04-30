# Contributing

## Workflow

1. **Open an issue first** — document what you're fixing or adding before writing code. One issue per change.
2. **Branch from `main`** — use the naming pattern `<type>/<short-description>` (e.g. `fix/vault-credential`, `feat/phase4-personas`).
3. **One fix per PR** — implement, commit, and open a PR for each issue separately before moving to the next.
4. **Reference the issue** — include `Closes #<number>` in your PR description.
5. **PRs target `main`** — direct pushes to `main` are not allowed. Branch protection is enforced for everyone including repo admins — no exceptions. If an emergency fix is needed, open a PR; it can be merged immediately without a reviewer.

## Commit messages

Follow the pattern used in this repo:

```
<type>: <short description>

<optional body explaining why, not what>
```

Types: `feat`, `fix`, `docs`, `refactor`, `chore`

## Code conventions

See [CLAUDE.md](CLAUDE.md) for:
- `ansible.platform` vs `ansible.controller` module preference
- Token cleanup rules
- Inventory pattern
- ansible.cfg design

## Testing

All playbook changes should be tested against a live RHDP **Ansible Product Demo** environment before opening a PR. Document how you tested in the PR description.

## Sensitive data

- Never commit credentials, tokens, or passwords
- Never commit inventory files under `inventories/rhdp-*/` (gitignored — `rhdp-sample-demo/` is the only exception)
- Never commit `docs/dev-environment.md` (gitignored)
