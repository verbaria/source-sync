# verbaria/source-sync

A GitHub Action that keeps a Verbaria server's **source** locale in sync with a
repository — the push counterpart to
[`verbaria/translate-sync`](https://github.com/verbaria/translate-sync).

On each run it:

1. Sets up **JDK 25** (configurable).
2. Downloads the **latest Verbaria CLI** release archive.
3. **Skips** if an open source pull request already exists.
4. **Backs up** the existing `verbaria-lock.json` (if any) before pushing.
5. Runs **`verbaria push --push-type source`** to upload the source locale.
6. Generates a **git commit message** and a **Markdown changelog** by diffing the
   old and new `verbaria-lock.json` (via `verbaria changelog`).
7. **Does nothing** when the source is unchanged (the lock writer ignores the
   volatile `generatedAt`, so an identical push produces no diff).
8. **Opens (or updates) a pull request** with the updated lock.

Typically triggered on pushes that touch the source files, so a change committed
to the repo is mirrored to Verbaria and the committed `verbaria-lock.json` stays
in step with the server.

## Usage

```yaml
# .github/workflows/source.yml
name: Sync source
on:
  push:
    branches: [ master ]
    paths: [ 'LOCALIZE-LIB/en_US/**' ]
  workflow_dispatch: {}

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: verbaria/source-sync@master
        with:
          server-url: https://translate.example.org/
          username: ${{ vars.VERBARIA_USERNAME }}
          api-key: ${{ secrets.VERBARIA_API_KEY }}
```

## Inputs

| Input                  | Required | Default                              | Description |
|------------------------|----------|--------------------------------------|-------------|
| `server-url`           | yes      | —                                    | Verbaria server URL. |
| `username`             | yes      | —                                    | Verbaria username. |
| `api-key`              | yes      | —                                    | Verbaria API key (use a secret). |
| `project-config`       | no       | `verbaria.json`                      | Path to the project config. |
| `working-directory`    | no       | `.`                                  | Where `verbaria.json` lives. |
| `push-type`            | no       | `source`                             | `source` or `both`. |
| `push-args`            | no       | `""`                                 | Extra args for `verbaria push`. |
| `client-repo`          | no       | `verbaria/verbaria`                  | Repo publishing the CLI archive. |
| `client-version`       | no       | `latest`                             | CLI release tag, or `latest`. |
| `client-download-url`  | no       | `""`                                 | Explicit `verbaria-*.tar.gz` URL (overrides repo/version). |
| `java-version`         | no       | `25`                                 | JDK version. |
| `java-distribution`    | no       | `temurin`                            | `actions/setup-java` distribution. |
| `branch`               | no       | `verbaria/source`                    | PR head branch. |
| `base-branch`          | no       | current branch                       | PR base branch. |
| `pr-title`             | no       | `chore(i18n): sync source`           | PR title. |
| `pr-labels`            | no       | `source`                             | Comma-separated PR labels. |
| `commit-author-name`   | no       | `verbaria-bot`                       | Commit author name. |
| `commit-author-email`  | no       | `verbaria-bot@users.noreply.github.com` | Commit author email. |
| `github-token`         | no       | `${{ github.token }}`                | Token for CLI download + PR. |

## Outputs

| Output    | Description |
|-----------|-------------|
| `changed` | `true` when the source changed and a PR was created/updated. |
| `skipped` | `true` when an open source PR already existed. |
| `pr-url`  | URL of the created/updated pull request. |

## Notes

- The lock diff drives both the commit message and the PR body, so an unchanged
  source push produces no commit and no PR.
- A re-push of identical source is a server-side no-op (no document/text-flow
  revision bump), so unchanged strings never flag translations for review.
- `github-token` needs `contents: write` and `pull-requests: write`; if the CLI
  release repo is private it must also be readable by the token.
- The CLI launcher selects its JVM from `VERBARIA_JRE`, which the action points at
  the JDK installed by `actions/setup-java`.
