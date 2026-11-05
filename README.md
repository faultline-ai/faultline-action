# Faultline GitHub Action

Faultline reviews every pull request for regressions: changes that could break existing behavior elsewhere in your codebase. This Action runs that review inside your own GitHub Actions runner and posts the result as one comment and one check on the pull request.

Using this Action requires a Faultline account.

## How it works

1. On each pull request, the Action proves your repository's identity to Faultline with a GitHub-issued OIDC token and your license token, and receives a short-lived authorization.
2. It downloads the Faultline analysis engine, then verifies its signature and checksum against the public keys pinned in your workflow before anything runs. A bundle that fails verification never runs.
3. The engine analyzes the change on your runner. If you supply an Anthropic API key, AI analysis calls go directly from your runner to Anthropic under your own account. Without a key, the run completes with deterministic analysis only and says so.
4. The Action posts the comment and check to your pull request, and uploads a record of the findings to Faultline for your hosted report and portal.

This repository contains only the Action's client code. The analysis engine is delivered separately at run time and is not part of this repository.

## What leaves your runner

By default (`redaction: paths-only`), no source code is uploaded. Findings identify code by file path, symbol name and line number, and every field that could quote code is removed on your runner before upload. As a second check, Faultline's servers refuse an upload whose findings include cited code or AI explanations, or whose report includes suggested fix or test code. Your pull request comment is rendered on your runner and always shows full code citations.

Setting `redaction: full` includes cited code snippets in your hosted report. Your organization's policy on our side must also allow `full`. If it does not, the upload is refused and the run has no hosted report; ask us to enable it.

Your Anthropic API key is passed only to the engine process on your runner. It is never uploaded.

## Usage

The Faultline GitHub App opens a pull request that adds this workflow for you. To set it up by hand:

```yaml
name: Faultline
on: pull_request
permissions:
  contents: read         # check out the repository
  id-token: write        # prove repository identity to Faultline
  pull-requests: write   # post the review comment
  checks: write          # publish the check
jobs:
  faultline:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # full history, so Faultline can find the merge base
      - uses: faultline-ai/faultline-action@v0
        env:
          FAULTLINE_TOKEN: ${{ secrets.FAULTLINE_TOKEN }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}   # optional
          FAULTLINE_LICENSE_URL: https://license.tryfaultline.com
          FAULTLINE_RESULTS_URL: https://api.tryfaultline.com
          FAULTLINE_PINNED_KEYRING: '{"keys":[...]}'   # Faultline's public signing keys, see below
```

- `FAULTLINE_TOKEN` (secret): your organization's license token, from the Faultline portal.
- `ANTHROPIC_API_KEY` (secret, optional): enables AI analysis under your own Anthropic account.
- `FAULTLINE_PINNED_KEYRING`: Faultline's public signing keys, written into the workflow file as a literal value. The engine download and Faultline's authorization are both checked against them. The App's setup pull request fills in the current value. If you set up by hand, copy the workflow from the "Add the workflow" step in the Faultline portal, which includes it. The same keys are published at `https://license.tryfaultline.com/.well-known/jwks.json`. When Faultline rotates its keys, the GitHub App opens a pull request that updates this line in `.github/workflows/faultline.yml`.

Pin to a release tag (`@v0.1.5`) or a commit SHA instead of `@v0` if you prefer exact versions.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `mode` | `analyze` | `analyze` reviews the pull request. `index` records package and symbol names for cross-repository analysis. `qa-verify` also runs your tests and checks a preview deployment. Availability depends on your plan. |
| `redaction` | `paths-only` | `paths-only` uploads no code. `full` includes cited snippets in your hosted report, if your organization's policy allows it. |
| `fail-on` | unset | Unset or `never`: findings never fail the check. `execution`: the check also fails when a regression was confirmed by actually running tests (requires `differential-testing` or `repro-synthesis`). At every setting, the check fails if the run itself fails; see [How results are reported](#how-results-are-reported). |
| `inline-comments` | `true` | Also post verified findings as inline review comments, at most 5 per pull request. |
| `intent-context` | `true` | Give the engine the pull request's title and description as context. The text is sent to Anthropic as part of the AI analysis, which runs when you supply an Anthropic API key. It is never uploaded to Faultline. |
| `org-index` | `true` | Take part in cross-repository analysis within your organization. Uploads package names, exported symbol names and import locations; never code. Set `'false'` to opt out. |
| `differential-testing` | `false` | Run your existing tests against the base and head of the change to confirm regressions. Requires the capability on your plan. |
| `repro-synthesis` | `false` | Have the AI write and run a small reproduction test for a finding. Requires the capability on your plan and an Anthropic API key. |
| `browser-qa` | `false` | In `qa-verify` mode, check the change's affected flows in a real browser against `preview-url`. |
| `qa-test-command` | unset | The command used to run a test file, for example `pnpm vitest run`. |
| `qa-build-command` | unset | `qa-verify` mode: an optional build or typecheck command. |
| `preview-url` | unset | `qa-verify` mode: a preview deployment to probe. |
| `qa-probe-paths` | `/` | `qa-verify` mode: comma-separated paths to probe on `preview-url`. |
| `channel` | `stable` | Engine release channel: `stable` or `canary`. |
| `github-token` | `${{ github.token }}` | Token used to post the comment and check. |

Your organization can set defaults and limits for some of these in the Faultline portal. Any setting that changes a run is stated on the check and the comment.

## Code execution

`differential-testing` and `repro-synthesis` run code on your runner. Both are off by default and need two switches: the input in your workflow, and the capability enabled on your plan. Faultline can never turn either on by itself, and can always turn them off.

Test code written by the AI is scanned before it runs and refused if it uses anything on a known list of network, process-starting, code-loading, or destructive file operations. It then runs with network proxy settings removed from its environment and a time limit on each run. The scan is a deny list, not a sandbox, so it cannot catch everything, and there is no operating-system sandbox. The main risk is prompt injection: content in your repository can influence the code the AI writes. Enable these features only where you are comfortable running test code from your repository and from the AI.

## How results are reported

- Findings are advisory. By default, findings never fail the check.
- The check fails when the run itself fails: an analysis step hit an error, or Faultline refused to run (for example, because the license token or the configuration is invalid). With `fail-on: execution`, it also fails on a regression confirmed by running tests.
- Whether a failed check blocks merging is decided by your repository's branch protection rules.
- A run that cannot certify its result shows a neutral check, never a green one.
- If uploading results to Faultline fails, your build is not failed. The check says what happened.
- If Faultline's licensing service is briefly unavailable, runs continue for a limited time using the authorization from your last successful run.
- A missing Anthropic key never fails a run. The run completes with deterministic analysis and says what adding the key unlocks.

## Network access

The runner needs outbound access to:

- `license.tryfaultline.com` and `api.tryfaultline.com` (Faultline, including the engine download)
- `api.anthropic.com` (only if you supply an API key)
- `*.actions.githubusercontent.com` (GitHub's OIDC tokens), `api.github.com`, `github.com`
- GitHub's Actions cache service (`results-receiver.actions.githubusercontent.com` and `*.blob.core.windows.net`), which GitHub-hosted runners reach by default. The Action caches the engine download and its last authorization there. If it is unreachable, runs still work, but cannot fall back on a cached authorization while Faultline's licensing service is unavailable.
- `huggingface.co` and its download CDN (`*.hf.co`), to download a small public model used for code-similarity analysis when it is not already on the runner. Nothing about your code is sent. If the download is blocked, a pull request review continues without that analysis; an `index` run fails. Air-gapped runners can provide the model files through `FAULTLINE_EMBEDDER_DIR` instead.

## Open-source software

This Action includes open-source software. Its licenses and notices are in [`dist/THIRD_PARTY_NOTICES.txt`](dist/THIRD_PARTY_NOTICES.txt).

## Security

Report security issues to security@tryfaultline.com.
