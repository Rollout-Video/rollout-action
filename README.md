# Rollout Video

Turn each shipped feature into a launch-ready demo or promo video, straight from your workflow.

Add one step after your deploy. The Action tells [Rollout](https://www.rollout.video) which commit
you shipped. Rollout reads what changed, decides whether there is a user-visible feature (a model
gate that ignores lockfile, test, docs and CI-only changes), and if so records your deployed app and
produces the video. Nothing heavy runs on your runner: no browser, no FFmpeg, no model keys, and no
secret to store. The Action authenticates with GitHub's OIDC token.

```yaml
- uses: Rollout-Video/rollout-action@v1
```

## Before you start

1. Install the **[Rollout Video GitHub App](https://github.com/apps/rollout-video)** on the
   repository.
2. Map the repository to a project at **rollout.video > Dashboard > GitHub**.

The Action refuses to start anything for a repository that has not done both. This is what stops
someone from spending your credits by copying your workflow.

## Usage

Record the app your workflow just deployed:

```yaml
name: Deploy
on:
  push:
    branches: [main]
permissions:
  contents: read
  id-token: write # lets the Action prove which repository and commit it runs for
jobs:
  deploy:
    runs-on: ubuntu-latest
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      - id: deploy
        run: ./scripts/deploy.sh # whatever you already do
  video:
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - id: rollout
        uses: Rollout-Video/rollout-action@v1
        with:
          url: ${{ needs.deploy.outputs.url }}
      - if: steps.rollout.outputs.should-create-video == 'true'
        run: echo "Video started for ${{ steps.rollout.outputs.feature-title }}"
```

Decide only, on pull requests (no credits spent, no run started):

```yaml
on: pull_request
permissions:
  contents: read
  id-token: write
jobs:
  preview-decision:
    runs-on: ubuntu-latest
    steps:
      - uses: Rollout-Video/rollout-action@v1
        with:
          dry-run: "true"
```

Pull requests from forks get no OIDC token from GitHub, so the Action cannot run for them.

## Inputs

| Input             | Default                     | Description                                                                                      |
| ----------------- | --------------------------- | ------------------------------------------------------------------------------------------------ |
| `url`             |                             | URL the workflow just deployed. Recorded instead of the project URL. Must be publicly reachable. |
| `base-sha`        | push `before` commit        | Compare from this commit. Falls back to the last commit Rollout handled for the repository.      |
| `policy`          | `auto`                      | `auto` (a model decides) or `always`. Can only narrow the repository's dashboard setting.        |
| `mode`            | `suggested`                 | `demo`, `promo`, or `suggested`. Accepted but ignored for now: the project's mode is used.       |
| `dry-run`         | `false`                     | Only decide; never start a run or spend credits.                                                 |
| `wait`            | `decision`                  | `none`, `decision` (return once decided) or `video` (poll until the run finishes or times out).  |
| `timeout-minutes` | `20`                        | Upper bound when `wait` is `video`.                                                              |
| `fail-on-error`   | `false`                     | Fail the step when Rollout errors. Off by default so a video never blocks a deploy.              |
| `api-url`         | `https://www.rollout.video` | Rollout origin (for staging). The OIDC token audience is this URL's origin.                      |

## Outputs

| Output                | Description                                                                                  |
| --------------------- | -------------------------------------------------------------------------------------------- |
| `should-create-video` | `true` or `false`.                                                                           |
| `category`            | `new-feature`, `improvement`, `fix`, `chore`, `docs`, `dependency`, `refactor` or `config`.  |
| `confidence`          | `0` to `1`.                                                                                  |
| `feature-title`       | The feature in a user's words, or empty.                                                     |
| `reason`              | Why, in plain words.                                                                         |
| `decided-by`          | `iris`, `gemini`, `policy` or `rule`.                                                        |
| `run-id`              | Rollout run id, empty when no run was started.                                               |
| `project-url`         | Link to the run on rollout.video.                                                            |
| `video-url`           | The exported MP4. Only set when `wait` is `video` and someone has exported it in the editor. |

Every run also writes a job summary with the decision and links.

## Behaviour

- **It never fails your job by default.** Any Rollout error ("App not installed", "repository not
  mapped", "not enough credits", a timeout) becomes a warning annotation and empty outputs. Set
  `fail-on-error: true` to turn errors into failures.
- Network errors, 5xx responses and 409 (Rollout is still handling the commit) are retried up to three times. Rollout is idempotent per
  commit, so a retry cannot start a second run.
- With `wait: video` the Action polls every 15 seconds (backing off) until the run finishes or
  `timeout-minutes` passes. A finished run links to the project, where you review and export the
  video; `video-url` is set only after the MP4 has been exported. Cancelling the job stops polling; the run itself continues on Rollout.
- If you also use Rollout's deployment webhook, one commit still produces one video: whichever
  trigger arrives first owns the commit, and the Action reports on that run.

## Troubleshooting

**"GitHub did not provide an OIDC token"** The job lacks permission to request one. Add this to the
workflow or to the job that runs the Action:

```yaml
permissions:
  contents: read
  id-token: write
```

Pull requests from forks never receive a token.

**404 "App not installed", or 403 "Not set up in Rollout"** Install the
[Rollout Video GitHub App](https://github.com/apps/rollout-video) on this repository and map it to a
project at rollout.video > Dashboard > GitHub. The 403 message names what is missing (for example
the installation is not linked to a Rollout account).

**401** Rollout did not accept the token, and the message says why. If it names the audience, check
`api-url`: the token is issued for that origin, which must be the Rollout deployment you are calling.

**422** The request was rejected, commonly because `url` is not publicly reachable. Preview URLs
behind a login or Deployment Protection cannot be recorded.

**429** Rate limited. Try again later.

**No video was started** Read the `reason` output or the job summary. Changes that only touch
lockfiles, tests, docs or CI are skipped on purpose. Use `policy: always` (if your dashboard policy
allows it) to skip the decision.

**`dry-run` on a pull request does nothing** It is meant to: it only reports the decision. Use the
outputs or the job summary.

## Pinning

`@v1` follows the latest `v1.x.y` release. Releases are tagged and immutable. For strict
supply-chain policies, pin to a full commit SHA instead:

```yaml
- uses: Rollout-Video/rollout-action@<full-commit-sha> # v1.0.0
```

The Action ships as a single bundled file with its dependencies inlined, so there is nothing to
install at run time.

## Privacy

The Action sends only the commit SHAs, the optional URL and the options above. Rollout reads the
repository itself through the GitHub App; no source code leaves your runner through this Action.
