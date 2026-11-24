# Contributing to linkwarden-cli

Thanks for helping out! This guide covers how to get the code, run it against a local Linkwarden, and get a change
merged. Contributions come in through **a fork and a pull request**: you don't need write access to this repository.

- [Before you start](#before-you-start)
- [Set up your fork](#set-up-your-fork)
- [Run it locally](#run-it-locally)
- [Make your change](#make-your-change)
- [Tests](#tests)
- [Commit messages](#commit-messages)
- [Open a pull request](#open-a-pull-request)
- [Keep your fork up to date](#keep-your-fork-up-to-date)

## Before you start

- **Open or find an issue first.** Every pull request closes an issue, so the problem is agreed on before anyone
  writes code. Search the [open issues](https://github.com/CerealKiller97/linkwarden-cli/issues), and if nothing fits,
  open one with the **Bug report** or **Feature request** form. For a small typo fix, the issue can be one line.
- **Security problems never go in a public issue or PR.** Follow [SECURITY.md](.github/SECURITY.md) instead.
- **Problems with Linkwarden itself** (the server or web app) belong in the
  [Linkwarden repository](https://github.com/linkwarden/linkwarden/issues).
- **A missing API endpoint goes into the SDK first.** Every API call goes through
  [linkwarden-go-sdk](https://github.com/CerealKiller97/linkwarden-go-sdk). If the endpoint you need isn't there, open
  a PR on the SDK, and the CLI picks it up with a version bump. Don't write raw HTTP calls in the CLI.

## Set up your fork

You need **Go 1.27+** (see `go.mod`), Git and, to test against a real instance, Docker with `curl`, `jq` and
`openssl`. Install [golangci-lint](https://golangci-lint.run/welcome/install/) v2 to lint locally.

1. Fork the repository on GitHub: **Fork** at the top of
   [CerealKiller97/linkwarden-cli](https://github.com/CerealKiller97/linkwarden-cli).
2. Clone your fork and add this repository as `upstream`:

   ```bash
   git clone https://github.com/<your-username>/linkwarden-cli.git
   cd linkwarden-cli
   git remote add upstream https://github.com/CerealKiller97/linkwarden-cli.git
   ```

   With the [GitHub CLI](https://cli.github.com), `gh repo fork CerealKiller97/linkwarden-cli --clone` does both.

3. Turn on the pre-commit hook, which keeps the generated command reference in `docs/cli/` in sync:

   ```bash
   git config core.hooksPath .githooks
   ```

4. Check that everything builds and passes:

   ```bash
   go build ./...
   go test ./...
   ```

## Run it locally

Run the CLI straight from source:

```bash
go run ./cmd/linkwarden --help
```

To try it against a real server without touching your own account, start a throwaway Linkwarden in Docker. The script
creates a test user and an API token:

```bash
scripts/dev-linkwarden.sh up              # start it (http://localhost:3000, user dev / devpassword)
eval "$(scripts/dev-linkwarden.sh env)"   # point LINKWARDEN_URL and LINKWARDEN_TOKEN at it
go run ./cmd/linkwarden links list
scripts/dev-linkwarden.sh down            # stop it; `reset` also deletes its data
```

Set `LINKWARDEN_VERSION=v2.11.0` (for example) before `up` to test against a specific Linkwarden release.

## Make your change

Create a branch in your fork, named after the change type and what it does:

```bash
git switch -c feat/highlights-update   # or fix/…, docs/…, refactor/…, test/…, chore/…, ci/…
```

Read the closest existing command before you write a new one, and copy its shape. `cmd/commands/tags/` is the
smallest complete group. The conventions that matter most:

- **One file per subcommand** in `cmd/commands/<group>/`, with an exported constructor that takes the App:
  `func Delete(app *app.App) *cobra.Command`. Name the parameter `app` and pass it explicitly.
- **ID flags:** the command's own entity gets a bare `--id` (or `--ids`). A flag pointing at another entity names it:
  `--collection-id`, `--link-id`.
- **Leave out `--id`, get a picker.** In a terminal, a missing ID opens a picker. With `--non-interactive` or no
  terminal, it fails and names the flag to pass.
- **Destructive commands** ask No / Yes first (No selected) and need `--yes` in scripts.
- **Output** goes to `app.Stdout()`, never `os.Stdout`, and respects `--json` and `-q`. Logs and errors go to stderr.
- **Errors** wrap with a lowercase verb phrase: `fmt.Errorf("get tag: %w", err)`.
- **Go style:** write parameter types out in full (`a string, b string`, never `a, b string`), and let `gofmt` and
  `goimports` format the code.

Before you push, run what CI runs:

```bash
go vet ./...
go test ./... -race
golangci-lint run
go run ./scripts/gendocs   # after changing any command, flag or help text; commit docs/cli/ with it
```

Also update `README.md` when you add a command group or change how something is used, and re-record the demo GIFs if
you changed output they show (see [`.github/assets/demo/`](.github/assets/demo/README.md)). You don't need to edit
`CHANGELOG.md`: it's generated from commit messages at release time.

## Tests

Every change to a command comes with tests. They follow a few rules:

- Test files use the external package: `package tags_test`. If a test needs an unexported name, add an
  `export_test.go` in the internal package.
- Each test starts with `t.Parallel()` and `assert := require.New(t)`, and calls `assert.X(...)`.
- Test names are PascalCase with no underscores: `TestDeleteAsksBeforeDeleting`.
- Split each test with `// Arrange`, `// Act` and `// Assert` comments (or `// Act & Assert` when the call is the check).
- Fake the API with `httptest.NewServer`, and build the App with `testutils.NewApp(server.URL, opts...)`, which also
  returns the output buffer.
- **Assert on what happened, not only that nothing failed:** how many requests were made, and each one's method, path,
  query, `Authorization` header and JSON body; what was printed; the error message (`assert.ErrorContains`). For a
  declined confirmation or invalid input, check that no request was sent.

For a destructive command, cover at least the happy path, an invalid flag value, a confirmed run and a declined run.
`CLAUDE.md` has a complete example test.

## Commit messages

Commits follow [Conventional Commits](https://www.conventionalcommits.org). The changelog and the next version number
are generated from them, so the type matters:

```text
feat(links): add --pinned filter to links list
fix: keep the spinner label visible until the request finishes
docs: explain token scopes in the README
feat!: rename --collection to --collection-id
```

| Type | Use for | Release |
| --- | --- | --- |
| `feat` | A new command, flag or behavior | minor |
| `fix` | A bug fix | patch |
| `docs` | README, guides, help text only | none |
| `refactor`, `test`, `chore`, `ci`, `style`, `perf` | Everything else | none |
| `!` after the type, or a `BREAKING CHANGE:` footer | Breaking commands, flags or JSON output | major |

The optional scope is the command group: `links`, `collections`, `tags`, `tokens`…

## Open a pull request

1. Push your branch to your fork:

   ```bash
   git push -u origin feat/highlights-update
   ```

2. Open a pull request from your branch against `main` in
   [CerealKiller97/linkwarden-cli](https://github.com/CerealKiller97/linkwarden-cli/pulls). GitHub shows a
   **Compare & pull request** button after the push, or run `gh pr create --repo CerealKiller97/linkwarden-cli`.
3. Fill in the pull request template:
   - Put `Closes #<issue number>` on the first line, one line per issue.
   - Tick the PR types, list the changes under Added / Changed / Fixed …, and say whether you added tests.
4. Leave **Allow edits by maintainers** ticked, so a maintainer can push small fixes to your branch.

CI runs the tests on Linux and macOS, `golangci-lint`, `gosec`, `govulncheck`, the `docs/cli/` check and ShellCheck.
A code owner reviews the PR once CI passes. Push more commits to the same branch to answer review comments; the PR
updates by itself.

## Keep your fork up to date

Before starting new work, and if your PR falls behind `main`:

```bash
git fetch upstream
git switch main
git merge --ff-only upstream/main
git push origin main
```

To update a branch that already has a PR open, rebase it on the new `main` and force-push:

```bash
git switch feat/highlights-update
git rebase upstream/main
git push --force-with-lease
```

## License

By contributing, you agree that your contributions are licensed under the same license as this project, found in
[LICENSE](LICENSE).
