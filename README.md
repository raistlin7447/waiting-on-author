# Waiting on Author

Waiting on Author adds a label to a pull request when the reviewers are waiting on its author, and takes it off once the author responds. Filter out the label to see the PRs that need a review.

It uses an AI model, through GitHub Copilot or Claude, to read the recent comments on each PR. Context matters in a reply, so the model looks at the whole recent conversation to decide. For example, a reply like "thanks, I'll fix that this week" keeps the label on, and pushing the fix takes it off.

It runs in your own workflow with your own tokens. It never checks out or runs code from a PR, and the model has no tools. PR titles and recent comments are sent to the model service you choose.

## Quick start

Pick one of the setups below and save it in your repository as `.github/workflows/waiting-on-author.yml`. It starts as a dry run, which only logs what it would change. Run it from the Actions tab, check the log, and set `dry-run: false` when you're happy with what you see.

<details>
<summary><b>Copilot, with a personal token</b></summary>

This uses your own Copilot subscription. Create a [fine-grained token](https://github.com/settings/personal-access-tokens/new) with the **Copilot Requests** account permission, and add it as a repository secret named `COPILOT_TOKEN`. Every request counts against your Copilot plan, and on the Free plan Copilot picks the model for you.

```yaml
name: Waiting on author

on:
  schedule:
    - cron: '7 * * * *'
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write
  checks: read

concurrency:
  group: waiting-on-author

jobs:
  sweep:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: raistlin7447/waiting-on-author@v1
        with:
          copilot-token: ${{ secrets.COPILOT_TOKEN }}
          # Set to false once the log looks right
          dry-run: true
```

</details>

<details>
<summary><b>Copilot, billed to an organization</b></summary>

This bills the organization's Copilot plan and needs no secret. An organization owner has to turn on the "Allow use of Copilot CLI billed to the organization" policy first. This setup hasn't been tested yet.

```yaml
name: Waiting on author

on:
  schedule:
    - cron: '7 * * * *'
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write
  checks: read
  copilot-requests: write

concurrency:
  group: waiting-on-author

jobs:
  sweep:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: raistlin7447/waiting-on-author@v1
        with:
          # Set to false once the log looks right
          dry-run: true
```

</details>

<details>
<summary><b>Claude</b></summary>

This uses a Claude subscription. Run `claude setup-token`, and add the token it gives you as a repository secret named `CLAUDE_CODE_OAUTH_TOKEN`. To pay per use with an Anthropic API key instead, pass the key in `anthropic-api-key`.

```yaml
name: Waiting on author

on:
  schedule:
    - cron: '7 * * * *'
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write
  checks: read

concurrency:
  group: waiting-on-author

jobs:
  sweep:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: raistlin7447/waiting-on-author@v1
        with:
          provider: claude
          claude-code-oauth-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          # Set to false once the log looks right
          dry-run: true
```

</details>

### What the first runs do

The first runs work through all your open PRs, oldest first, 20 per run. After that, each run only looks at PRs with new activity. The `held` count in the log tells you how many are still waiting their turn. To get through a big backlog faster, run the workflow by hand or raise `max-judgments`, and raise `timeout-minutes` with it. Dry runs and real runs keep track of progress separately, so your first real run starts again from the oldest PR.

Progress is saved in the Actions cache. If the workflow doesn't run for 7 days, GitHub deletes it, and the next run starts over with every open PR.

### Requirements

It needs a Linux or macOS runner with `gh` and `python3`. Copilot also needs `pip` and Python 3.11 or newer, and Claude needs `npm`. `ubuntu-latest` has all of these.

## How it decides

For each open PR, the Action works out whether the next step is the author's. A typical PR goes like this.

1. **The author opens a PR.** No label. Nobody has asked them for anything yet, so the Action doesn't ask the model.
2. **A reviewer asks for a change.** Once the conversation has been quiet for 30 minutes, the model reads the recent comments and sees a request the author hasn't answered. The label goes on. Waiting for things to go quiet means a quick back-and-forth is judged once, at the end.
3. **The author replies "thanks, I'll fix that this week".** The request is still open, so the label stays.
4. **The author pushes a fix.** The label comes off. A push after the label went on, with no team comment since, counts as the author's answer, so the model isn't asked.
5. **The reviewer says it's still broken.** The model reads the conversation again and puts the label back on. If the author had asked the reviewer a question instead, the next step would be the reviewer's, and the label would stay off.

The reviewer here is anyone on the team, which means your repository's owners, members and collaborators unless you change `team-associations` or `team-users`. Comments from bots and from anyone else don't count.

### Safeguards

A wrong "waiting" hides a PR from reviewers, which is worse than a missing label. So when in doubt, the Action leaves the label off.

- The model has to answer "waiting" three times in a row. If it changes its mind, the PR counts as not waiting.
- If someone adds or removes the label by hand, the Action leaves it alone until there's new activity.
- Drafts never get the label, since their authors are still working on them. `label-drafts` changes that.

### When something goes wrong

If a PR can't be judged or its label can't be changed, the next few runs try it again, up to `failure-retries` times, and then move on. If every PR in a run fails, which usually means an expired token or a model service outage, the run fails without moving forward, so nothing gets skipped while you fix it.

## Inputs

Every input has a default, so most setups only set a provider and its token. Lists are comma-separated, and logins work with or without `@`. A value the Action can't use stops the run before it does anything, with an error that says what the input accepts.

### Provider and tokens

| Input | Default | Description |
|---|---|---|
| `provider` | `copilot` | `copilot` or `claude` |
| `copilot-token` | (`github-token`) | Token with the Copilot Requests permission |
| `claude-code-oauth-token` | | Token from `claude setup-token` |
| `anthropic-api-key` | | API key to use instead of the OAuth token |
| `github-token` | `github.token` | Token that reads PRs and changes labels. Needs Pull requests (write) and Checks (read) |
| `claude-code-version` | `2.1.318` | Claude Code CLI version to install |
| `copilot-sdk-version` | `1.0.28` | Copilot SDK version to install |

### Which PRs, and the label

| Input | Default | Description |
|---|---|---|
| `repository` | this repository | Repository to label, as `owner/name` |
| `pr-filter` | | Search terms that narrow which open PRs are looked at, in the same syntax as GitHub's PR search |
| `label` | `waiting on author` | Label name, created if missing |
| `label-color` | `fbca04` | Color of the label, if the Action creates it |
| `label-drafts` | `false` | Label draft PRs too |

`pr-filter` takes the same terms you'd type into the search box on your repository's pull requests page, such as `base:main -label:"on hold"` to skip parked PRs and only look at ones into `main`. A PR that stops matching keeps the label it has.

### Who's on the team

| Input | Default | Description |
|---|---|---|
| `team-associations` | `OWNER,MEMBER,COLLABORATOR` | Author associations that count as the team. Empty means only `team-users` |
| `team-users` | | More logins that count as the team, bots included, such as `alice, @release-bot` |
| `ignored-users` | | Logins to ignore, such as `ci-helper` for a person's account that a bot posts from. Bot accounts are ignored unless they're in `team-users` |
| `command-prefix` | | Skip comments that start with this, such as `/bot` |

### How the model judges

| Input | Default | Description |
|---|---|---|
| `model` | `claude-sonnet-5-5` for Claude, Copilot's choice for Copilot | Model ID, such as `claude-sonnet-5.5` on Copilot or `claude-sonnet-5-5` on Claude. On Claude, a family name like `sonnet` means its newest model |
| `effort` | provider's default | Reasoning effort, such as `low` or `high`. The levels depend on the model |
| `confirmations` | `2` | Extra calls that must also answer "waiting" before it counts |
| `quiet-minutes` | `30` | Minutes a PR must be quiet before it's judged |
| `context-messages` | `6` | Recent messages the model reads |
| `max-text` | `2000` | Characters kept from each message |
| `prompt-file` | [prompt.md](prompt.md) | File in your repository with your own instructions, such as `.github/waiting-on-author.md`. See below |
| `model-timeout` | `300` | Seconds to wait for each call |

Your file replaces the built-in instructions completely, so start from a copy of [prompt.md](prompt.md) and change what you need. For example, you could add that a PR whose author was asked to sign your contributor agreement is waiting on them until they say they did. Wherever the file says `{project}`, the model sees the repository name, so "You track whether {project} pull requests" becomes "You track whether my-app pull requests" on `you/my-app`.

The Action reads the file from the runner, so check out your repository in a step before it:

```yaml
    steps:
      - uses: actions/checkout@v7
        with:
          persist-credentials: false
      - uses: raistlin7447/waiting-on-author@v1
        with:
          prompt-file: .github/waiting-on-author.md
```

### Runs and progress

| Input | Default | Description |
|---|---|---|
| `dry-run` | `true` | Only log what would change |
| `since` | | Time to start from instead of the saved progress, in UTC, such as `2026-11-01T00:00:00Z` |
| `lookback-hours` | `0` | Hours to look back without saved progress. `0` means all open PRs |
| `max-judgments` | `20` | PRs judged per run, with the rest left for the next. `0` means no limit |
| `failure-retries` | `3` | Runs that retry a failed PR before moving on |
| `github-tries` | `3` | Attempts per GitHub search |
| `github-retry-delay` | `10` | Seconds between attempts |
| `log-reasons` | `false` | Log the model's reason for each PR. Logs are public on a public repository |

### How much history is fetched

These cap the most recent items fetched for each PR. The model only reads what's fetched, so raise them along with `context-messages`.

| Input | Default | Description |
|---|---|---|
| `max-comments` | `30` | Comments |
| `max-reviews` | `20` | Reviews |
| `max-review-comments` | `20` | Inline comments per review |
| `max-commits` | `10` | Commits |
| `max-label-events` | `30` | Label changes |

## Outputs

Each run reports these totals in its log, its job summary and its step outputs.

| Output | Description |
|---|---|
| `added`, `removed` | PR numbers, comma-separated |
| `judged`, `held`, `failures` | PR counts |
| `model-calls` | Model calls made |
| `input-tokens`, `output-tokens` | Tokens used |
| `cost-usd` | Claude's cost at list price. On a subscription, this is an estimate |
| `ai-credits` | Copilot AI credits used |

A call uses about 0.3 Copilot AI credits with Sonnet 5.5. Each PR takes one call, plus up to `confirmations` more when the answer is "waiting".

## Closing abandoned PRs

This Action never closes anything. To close PRs whose authors never come back, point [`actions/stale`](https://github.com/actions/stale) at the label:

```yaml
- uses: actions/stale@v11
  with:
    only-pr-labels: waiting on author
    days-before-pr-stale: 90
    days-before-pr-close: 14
    # Without these two, actions/stale also closes old issues
    days-before-issue-stale: -1
    days-before-issue-close: -1
```

## Running it locally

To run the tests, or a dry run against a repository from your own machine:

```bash
python3 -m unittest discover -s tests
pip install github-copilot-sdk
WAIT_LABEL_REPO=owner/name WAIT_LABEL_LOG_REASONS=true python3 wait_label.py --since 2026-10-27T00:00:00Z
```

It changes nothing without `--apply`. It uses your `gh` login for GitHub, and reads the provider's token from `WAIT_LABEL_COPILOT_TOKEN`, or from `CLAUDE_CODE_OAUTH_TOKEN` with `WAIT_LABEL_PROVIDER=claude`.
