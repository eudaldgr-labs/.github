# .github

Org-wide defaults for [eudaldgr-labs](https://github.com/eudaldgr-labs), following the layout of [home-operations/.github](https://github.com/home-operations/.github). Everything here applies to every repository in the organisation: GitHub reads some of it from this repository by name, and the other repositories pull in the rest explicitly.

## Composite actions

Shared CI steps, each versioned and tagged on its own. Reference them by commit SHA with a `# <action>-vX.Y.Z` comment; `renovate-config.json` updates the pair from this repository's per-action tags, and a branch or tag reference would trip zizmor's `unpinned-uses` audit in the calling repository.

| Action                                   | Purpose                                                                                   |
| ---------------------------------------- | ----------------------------------------------------------------------------------------- |
| [`collect-patch`](actions/collect-patch) | Runs a command on a checkout without write credentials and uploads the changes as a patch |
| [`push-patch`](actions/push-patch)       | Commits that patch as a GitHub App bot, pushes it and optionally opens a pull request     |

Each has its own changelog and `<action>-<version>` tag, so one action's release never moves the other's version.

### collect-patch / push-patch

Mend-hosted Renovate cannot run `postUpgradeTasks`. A dependency bump that needs a generated change (a refreshed remediation list, an APK epoch) is therefore regenerated in a workflow and pushed back. Scheduled jobs use the same pair to open bot pull requests.

- `collect-patch` checks out the commit, runs a command under `mise exec` and uploads the changes to `paths` as a patch. It never sees a write token.
- `push-patch` mints a GitHub App token, applies the patch, stages only `paths`, commits as the App's bot user and pushes. With `pr-title`, it opens or updates a pull request.

The command runs dependency code, so keeping it out of the job that holds the token means nothing it leaves running can read the token.

```yaml
jobs:
  regenerate:
    if: >-
      ${{ github.event.pull_request.user.login == 'renovate[bot]'
      && github.event.pull_request.head.repo.full_name == github.repository }}
    runs-on: ubuntu-26.04
    permissions:
      contents: read
    outputs:
      changed: ${{ steps.collect.outputs.changed }}
    steps:
      - id: collect
        uses: eudaldgr-labs/.github/actions/collect-patch@<sha> # collect-patch-v1.0.0
        with:
          run: just generate
          paths: |
            */melange.yaml

  push:
    needs: regenerate
    if: ${{ needs.regenerate.outputs.changed == 'true' }}
    runs-on: ubuntu-26.04
    permissions:
      contents: read
    steps:
      - uses: eudaldgr-labs/.github/actions/push-patch@<sha> # push-patch-v1.0.0
        with:
          client-id: ${{ secrets.BOT_CLIENT_ID }}
          private-key: ${{ secrets.BOT_APP_PRIVATE_KEY }}
          paths: |
            */melange.yaml
```

- Commits come from a GitHub App, not `GITHUB_TOKEN`: a push with `GITHUB_TOKEN` starts no workflow run, so required checks would never report on the new commit.
- Renovate treats a branch with commits from another author as modified and stops rebasing it. Once the App exists, add its bot email (`<id>+<slug>[bot]@users.noreply.github.com`) to `gitIgnoredAuthors` in `renovate-config.json`.
- Renovate rebuilds its branch when it rebases, dropping the regenerated commit, so the workflow runs again. A run on an already regenerated head changes nothing and stops.
- The push is made on the pull request head without force, unless `force: true` is set for a bot-owned branch. A concurrent push makes the run fail and the next run takes over.

#### GitHub App

Create an App owned by the organisation with _Contents: read and write_ and _Pull requests: read and write_, install it on the repositories that use these actions and store `BOT_CLIENT_ID` and `BOT_APP_PRIVATE_KEY` as organisation secrets. The same App should then sign release-please's token in `.github/workflows/release-please.yaml`.

## Shared configuration

| Path                   | Purpose                                                                                   |
| ---------------------- | ----------------------------------------------------------------------------------------- |
| `lefthook.common.toml` | Pre-commit hooks every repository extends as a lefthook remote, pinned to `main`          |
| `renovate-config.json` | Renovate preset every repository extends as `local>eudaldgr-labs/.github:renovate-config` |
| `.github/workflows`    | CI for this repository and release-please for the actions                                 |

Because repositories track `lefthook.common.toml` at `main`, a hook added here takes effect everywhere on the next commit. A hook that shells out to a tool needs that tool pinned in each repository's `.mise/config.toml` first.

`renovate-config.json` is policy only: commit format, labels, groups, release ages and automerge. Managers that only one repository needs live in that repository's `.renovaterc.json5`.

## Releases

release-please tags each action from conventional commits (`collect-patch-v1.0.0`, `push-patch-v1.0.0`). The first tags are created by hand, because the manifest already names `1.0.0`.
