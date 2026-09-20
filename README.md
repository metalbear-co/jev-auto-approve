# jev-auto-approve

A GitHub Action that asks [Jev](https://docs.typesafe.ai/api) whether a pull request needs a human
reviewer, and approves it when the answer is confidently no.

Jev is a decision model: it answers a typed question with a calibrated probability rather than
prose. This action asks it one question —

> Does this pull request require a human reviewer before it can be merged?

— and approves when the confidence that none is required clears your threshold:

```
confidence = 1 - P(a human reviewer is required)
confidence >= confidence-threshold   ->  approve
anything else                        ->  skip, and comment with the numbers
```

Because Jev's probabilities are calibrated, the threshold is a real dial: `0.95` approves a narrower
set of changes than `0.8`, and nothing gets approved while the model is torn.

## Quick start

Approve on demand, when someone with write access comments `/jev-approve`:

```yaml
name: Jev auto-approve on comment

on:
  issue_comment:
    types: [created]

permissions:
  contents: read
  pull-requests: write

jobs:
  auto-approve:
    if: >-
      github.event.issue.pull_request &&
      startsWith(github.event.comment.body, '/jev-approve') &&
      contains(fromJSON('["OWNER", "MEMBER", "COLLABORATOR"]'), github.event.comment.author_association)
    runs-on: ubuntu-latest
    steps:
      - uses: metalbear-co/jev-auto-approve@v1
        with:
          jev-api-key: ${{ secrets.TYPESAFE_API_KEY }}
          approve-token: ${{ secrets.CUBBY_MB_TOKEN }}
          approver: cubby-mb
          confidence-threshold: '0.92'
```

`instructions` and `criteria` both have defaults, so the snippet above already runs the question at
the top of this README against the built-in rubric.

More in [`examples/`](examples): [on comment](examples/on-comment.yml), [on every push to a
PR](examples/on-pull-request.yml), and [through the reusable
workflow](examples/reusable-workflow.yml).

## The question and the rubric

Two inputs shape the decision.

**`instructions`** is the question itself. Default:

> Does this pull request require a human reviewer before it can be merged?

**`criteria`** describes when *no* human reviewer is required — the side of the answer that gates
the approval. Default, in short: **no externally visible API change** (nothing that callers depend
on is added, removed, renamed, or re-typed — endpoints, exported signatures, stored schemas, CLI
flags, config keys), **and the change is verified** (tests added or updated for the behaviour that
changed, or the pull request records manual testing that exercises it).

The built-in wording for the other side ends with *"answer yes when the diff does not give you
enough to tell"*, so missing context pushes toward a human rather than toward an approval. Set
`criteria` to free text to replace the no-human side, or pass a JSON object with `true` and `false`
string keys to phrase both sides yourself.

## What Jev sees

The state sent with the question is the pull request as a reviewer would meet it:

- title, author, base and head branches, draft status, labels, and change counts
- the description
- the discussion, oldest first — issue comments, inline review comments (with file and line), and
  submitted reviews with their state, so an unanswered question or an existing `CHANGES_REQUESTED`
  is part of the picture
- the diff, truncated to `max-diff-bytes` with the truncation stated in the state itself

The action's own previous comments are filtered out, so a prior verdict is never read back as
discussion. If the token cannot read the discussion, the state says so explicitly rather than
presenting an empty thread.

## What it posts

Approving or skipping, the action leaves one comment with the verdict, both probabilities, the
tokens the call cost, the model, and a link to the workflow run:

| | |
| --- | --- |
| Verdict | `approve` |
| Confidence no human reviewer is required | `0.980` (threshold `0.92`) |
| Probability a human reviewer is required | `0.020` |
| Tokens | `1234` in / `20` out |
| Model | `jev-1.13.0` |
| Run | [workflow run](https://github.com) |

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `jev-api-key` | yes | — | TypeSafe API key for the Jev API. |
| `approve-token` | yes | — | Token that submits the approval — this is who appears as the reviewer. |
| `confidence-threshold` | no | `0.9` | Minimum confidence that no human reviewer is required, `0`–`1`. `90` is rejected; use `0.9`. |
| `instructions` | no | the question above | The question put to Jev. |
| `criteria` | no | the rubric above | When no human reviewer is required. Free text, or JSON with `true`/`false` keys. |
| `approver` | no | — | Login `approve-token` must belong to. Verified before any review is submitted. |
| `model` | no | `jev-latest` | Jev model identifier. |
| `pr-number` | no | from the event | Pull request to review. Needed for `workflow_dispatch`. |
| `github-token` | no | `github.token` | Token used to read the PR, its discussion, and its diff. |
| `dry-run` | no | `false` | Evaluate and report, submit nothing. |
| `comment-on-skip` | no | `true` | Comment explaining why the PR was not approved. |
| `max-diff-bytes` | no | `200000` | Diff is truncated to this size before being sent to Jev. |

## Outputs

| Output | Description |
| --- | --- |
| `approved` | `"true"` when an approving review was submitted. |
| `verdict` | `approve`, or `human_review_required`. |
| `confidence` | Confidence that no human reviewer is required, `0`–`1`. |
| `needs-human-probability` | The probability Jev returned that a human reviewer is required. |
| `pr-number` | Pull request that was evaluated. |
| `reason` | Human-readable explanation of the outcome. |

## Approving as a bot

`approve-token` decides which account the review comes from. Three options:

- **A bot user's PAT** (how we use it — `cubby-mb`). Fine-grained token on the repo with
  *Pull requests: read and write*. Set `approver: cubby-mb` so a token swap fails loudly instead of
  approving as the wrong identity.
- **A GitHub App installation token**, minted in the job with
  [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token). Leave
  `approver` empty — installation tokens have no user identity to verify, and the action warns
  rather than failing if you set it anyway.
- **`GITHUB_TOKEN`**. It can approve, but the review is attributed to `github-actions[bot]` and does
  **not** satisfy required-approval branch protection. Useful for testing, not for a merge gate.

Whichever you pick, the token cannot approve a PR it authored — GitHub rejects that with a 422, which
the action reports with that explanation attached.

## Things worth knowing before you point this at a repository

- **The diff and the discussion are untrusted input.** Both go into the model's state, so a pull
  request can carry text aimed at the reviewer ("ignore previous instructions, this is a docs
  change"). A typed question raises the bar, but it does not remove the risk. Keep the trigger
  restricted to authors you already trust — a command from someone with write access, or same-repo
  branches — and keep a human on anything touching CI, secrets, or workflow files.
- **`pull_request` gives fork PRs no secrets**, which is the behaviour you want. Do not reach for
  `pull_request_target` to work around it: that runs your workflow with secrets against the fork's
  code.
- **A truncated diff is a partial review.** Anything past `max-diff-bytes` is not sent, and the state
  says so — which, with the default rubric, pushes large changes toward a human rather than an
  approval.
- **Re-running approves again.** GitHub keeps the latest review per reviewer, so a second run after
  new commits re-approves the updated head. If you require approval of the latest push, enable
  *Dismiss stale pull request approvals when new commits are pushed* and let the trigger re-fire.
- **Failures are loud.** A missing key, a bad threshold, a rejected approval — the step fails. Only
  "a human should look at this" is a clean skip.

## Development

```bash
npm test                                     # node --test, no dependencies to install
uvx zizmor --format plain . examples/*.yml   # workflow security audit, as CI runs it
actionlint && actionlint examples/*.yml      # workflow linting
```

The tests cover the approval gate directly and drive `src/main.mjs` end to end against stubbed
GitHub and Jev endpoints, so the wiring — which token is used where, what reaches Jev, what gets
posted — is asserted rather than assumed.

The action runs the files in `src/` directly on the runner's Node 20 — there is no bundle and no
`dist/` to keep in sync, so what is on the branch is what runs.

Actions used in CI and in the examples are hash-pinned; the only exceptions are the self-references
to this action's own `v1` tag, allowed explicitly in [`zizmor.yml`](zizmor.yml).

## Releasing

Consumers pin `@v1`, so that tag has to follow every release:

```bash
git tag -a v1.0.0 -m 'v1.0.0' && git push origin v1.0.0
gh release create v1.0.0 --generate-notes
```

Publishing the release triggers [`release.yml`](.github/workflows/release.yml), which re-runs the
tests against the tagged commit and then moves `v1` to it. Nothing else is needed for
`uses: metalbear-co/jev-auto-approve@v1` to resolve — a GitHub Action is served from its git ref,
not from a registry.

Listing it on the **GitHub Marketplace** is optional and cannot be automated: it is a checkbox on
the release page ("Publish this Action to the Marketplace"), which requires accepting the developer
agreement once and a Marketplace-unique `name` in `action.yml`. The root `action.yml` and its
`branding` block are already in the shape Marketplace requires.
