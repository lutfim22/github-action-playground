# github-action-playground

Manual workflows that create pull requests as `github-actions[bot]`, allowing the user who starts the workflow to review and approve the resulting PR according to repository rulesets and branch protection.

## Current status and testing

Both workflows have been merged into `main`. The maintainer has confirmed that the bot workflow works. The `test/readme-current-status` branch contains documentation updates for testing PR creation after the initial setup.

To test the workflow without a PAT:

1. Open **Actions → Create PR (GitHub Actions bot) → Run workflow**.
2. Select **Use workflow from → test/readme-current-status**.
3. Leave `base_repo`, `head_repo`, `head_branch`, `title`, and `body` empty; use `base_branch=main` and `draft=false`.
4. Run the workflow. The expected result is a PR targeting `main`, authored by `github-actions[bot]`, with the title **docs: update current status and action test instructions**.
5. Open the PR link from the run summary to review and approve it with your account. Run the workflow again with the same inputs to check that it returns the existing open PR.

These steps cover PR creation, the automatic title, and reuse of an existing PR within the same repository. Fork sources and workflow changes on a branch require separate tests. Keep the test branch unmerged until testing is complete: the source must contain additional commits relative to the base.

## Primary workflow: no PAT required

File: `.github/workflows/create-pr.yml` — **Create PR (GitHub Actions bot)**.

1. For the initial setup, merge the workflow into the **default branch** (`main` in this repository).
2. Enable **Settings → Actions → General → Workflow permissions → Allow GitHub Actions to create and approve pull requests**. Organization policies may restrict this setting. The workflow requests `contents: read` and `pull-requests: write`; there is no need to grant write access to every workflow.
3. Open **Actions → Create PR (GitHub Actions bot) → Run workflow**.
4. Select a branch in **Use workflow from**, provide any input overrides, and run the workflow. The PR link appears in the run summary.

| Input | Default / behavior |
| --- | --- |
| `base_repo` | The repository running the workflow. Bot mode requires this repository as the target. |
| `base_branch` | `main`; can be changed to another branch in the target repository. |
| `head_repo` | The repository running the workflow; accepts `owner/fork` from the same fork network. |
| `head_branch` | The branch selected in **Use workflow from**. Set this explicitly for a fork branch or another source branch. |
| `title` | The subject of the first commit in the base-to-head comparison: a commit present in head but absent from base, in the GitHub API's chronological order. This is not the first commit in the repository's entire history. |
| `body` | Empty. |
| `draft` | `false`. |

Same-repository example: select `feature/example` in **Use workflow from** and leave the other inputs empty or at their defaults. This creates a PR from `feature/example` to `main` in this repository. The selected branch must already contain the workflow file.

Fork-source example: run the workflow in the target repository, set `head_repo=someone/github-action-playground` and `head_branch=feature/example`, and leave `base_repo` empty with `base_branch=main`.

An existing open PR for the same source repository, source branch, and target branch is reused. The workflow does not update its title or body. A source with no additional commits is rejected. GitHub determines whether the changes can form a PR.

## Forks and token limitations

`GITHUB_TOKEN` is scoped to the repository running the workflow. Permission to create or approve PRs does not grant write access to every fork or repository owned by the user.

- Public fork sources can be used within the same fork network. Private sources must be readable by the token; the built-in token does not automatically receive access to other private repositories.
- To use another fork as the **base repository** without a PAT, install and run the workflow in that target fork. Enable Actions there if it is disabled.
- Both repositories must belong to the same fork network. Repositories with matching contents but no fork relationship are unsupported.
- The workflow does not check out or execute source code from the fork. Inputs are passed through environment variables rather than interpolated into JavaScript or shell code.

### Forking without a second personal account

You can create a fork in an **organization** you manage using your existing GitHub account. Open **Fork → Create a new fork**, then select the organization as the **Owner**. The organization must allow repository and fork creation, and you must have the necessary permissions. If needed, create an organization first; a second personal account is unnecessary.

The original repository already belongs to `lutfim22`, so a regular fork cannot be created again in the same personal namespace within that fork network. Creating a repository from a template or pushing a mirror does not establish the fork relationship required for cross-fork testing.

Once the public fork exists, create a branch with an additional commit in the fork. Run the bot workflow in the target repository with `head_repo=ORG_NAME/FORK_NAME` and `head_branch=BRANCH_NAME`. To test the opposite direction, enable Actions and run the workflow in the organization fork as the target repository.

## Branch dropdowns

GitHub Actions does not support dynamic API-backed dropdown options for `workflow_dispatch.inputs`. A `type: choice` input only accepts a static list in YAML.

**Use workflow from** provides a branch/tag selector for the current repository. The workflow uses the selected branch as the default `head_branch`. Base branches and fork branches use text inputs. The workflow does not rewrite YAML to refresh dropdown options.

## Testing changes before merging

`workflow_dispatch` must first be registered on the default branch. The initial setup requires a merge into `main`; the workflow does not merge itself automatically.

After the initial setup:

1. Create and push a test branch containing the YAML changes and an additional commit.
2. Run the workflow using **Use workflow from → your test branch**. Jobs and steps from that ref are executed, allowing implementation changes to be tested before merging.
3. Leave `head_branch` empty to create a PR from the test branch to `main`, or specify the source explicitly.

The workflow selector does not select source forks. For a fork source, select a workflow ref from the repository running the workflow, then provide `head_repo` and `head_branch`.

The UI input form may follow the definition on the default branch. Update the default branch when changing input names or schemas so that the form matches. Use registered inputs when testing implementation changes on another branch.

## Optional: separate PAT workflow

File: `.github/workflows/create-pr-pat.yml` — **Create PR (PAT)**. The bot workflow has no hidden PAT fallback.

Add the repository secret `PR_CREATOR_PAT`, then run this workflow. Set `base_repo` to another repository authorized by the token. A fine-grained PAT needs **Pull requests: Read and write** on the target and read access to the source and target repositories and branches. The token owner must also have the required repository access. Resource-owner scope and organization policies may limit cross-fork choices; a classic PAT requires scopes appropriate to the repository's visibility.

**A PR created with a PAT is authored by the PAT owner.** Using your own PAT still prevents you from approving that PR yourself. To preserve independent approval while using credentials across repositories, use a separate bot account or a GitHub App with an installation token and appropriate repository access. GitHub App setup is outside this implementation.

## PR review and CI

Approving a PR review and approving a CI workflow run are separate actions. The bot creates the PR, and a human reviews it according to repository rules. These workflows do not automatically approve or merge PRs.

Current GitHub documentation describes `pull_request` events of type `opened`, `synchronize`, or `reopened` generated using `GITHUB_TOKEN` as potentially creating runs in an **approval-required** state. A user with write access can select **Approve workflows to run**. On hosts or versions without that behavior, events generated by the built-in token generally do not trigger subsequent workflows; `workflow_dispatch` and `repository_dispatch` are exceptions. A successfully created PR does not establish that CI has run.

## References

- [Running a workflow manually](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow)
- [workflow_dispatch inputs](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#onworkflow_dispatchinputs)
- [GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token)
- [Events generated by GITHUB_TOKEN, GitHub documentation source](https://github.com/github/docs/blob/main/data/reusables/actions/actions-do-not-trigger-workflows.md)
- [Create pull request API and cross-repository heads](https://docs.github.com/en/rest/pulls/pulls#create-a-pull-request)
