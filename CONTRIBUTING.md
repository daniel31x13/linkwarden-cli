# Contributing

Thanks for helping out! Changes come in through a fork and a pull request. Everyone taking part follows the
[Code of Conduct](CODE_OF_CONDUCT.md).

## 1. Start with an issue

Every pull request closes an issue. Find one in [issues](https://github.com/CerealKiller97/linkwarden-cli/issues) or
open a new one. Security problems go through [SECURITY.md](.github/SECURITY.md), never a public issue.

## 2. Fork and set up

You need Go 1.27+.

```bash
gh repo fork CerealKiller97/linkwarden-cli --clone   # or fork on GitHub and git clone your fork
cd linkwarden-cli
git config core.hooksPath .githooks                  # keeps docs/cli/ in sync on commit
go test ./...
```

To try your change against a throwaway Linkwarden in Docker:

```bash
scripts/dev-linkwarden.sh up
eval "$(scripts/dev-linkwarden.sh env)"
go run ./cmd/linkwarden links list
```

## 3. Make your change

- Work on a branch: `git switch -c feat/short-name` (or `fix/…`, `docs/…`).
- Copy the shape of a nearby command. `cmd/commands/tags/` is a small, complete example.
- Add tests next to your code. See the **Tests** section of [CLAUDE.md](CLAUDE.md) for the style.
- If the API endpoint you need is missing, add it to [linkwarden-go-sdk](https://github.com/CerealKiller97/linkwarden-go-sdk) first.

Before pushing, run:

```bash
go test ./...
golangci-lint run
go run ./scripts/gendocs   # after changing a command or flag
```

## 4. Commit and open the PR

Write commit messages as [Conventional Commits](https://www.conventionalcommits.org), since the changelog is built
from them: `feat(links): …`, `fix: …`, `docs: …`, or `feat!: …` for a breaking change.

Push to your fork and open a pull request against `main`. Fill in the template and put `Closes #<issue>` on the
first line.

By contributing, you agree your work is licensed under this project's [LICENSE](LICENSE).
