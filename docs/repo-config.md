# Faultline — Repository Configuration File (`.faultline.yml`)

A repository can steer Faultline with one optional file at its root, `.faultline.yml`. It has three settings: paths to leave out, context for the AI review, and the lowest severity to post on the pull request.

> **Requires Faultline Action v0.1.7 or later.** A workflow that uses `faultline-ai/faultline-action@v0` already has it. On a workflow pinned to v0.1.6 or earlier the file is ignored and the run does not mention it.

**You do not need this file.** Without it, Faultline behaves exactly as it does today. Add it when a repository has paths that should never be reviewed (generated code, vendored code, snapshots) or when findings below a certain severity are noise for your team.

```yaml
# .faultline.yml — at the root of the repository

# Paths Faultline leaves out. Nothing is reported on these files.
ignore:
  - "vendor/"
  - "src/generated/**"
  - "*.snap"

# Context for the AI review, in plain language. At most 2,000 characters.
instructions: |
  We are migrating the billing module from v1 to v2.
  Removing v1 serializer fields under src/billing/legacy is intended.

# Lowest severity posted on the pull request: low, medium, or high.
min_severity: medium
```

## Where it goes and when it is read

- The file name is exactly `.faultline.yml`, at the root of the repository. A file with another name (`.faultline.yaml`, `faultline.yml`) or in a subdirectory is not read.
- It is read on every pull-request run, from the pull request's own head commit. A change to the file takes effect in the pull request that makes it.
- It applies to `analyze` runs, including the analysis part of `qa-verify`. It does not apply to `index` or `backtest` runs.

## The three settings

### `ignore`

A list of path patterns. Files that match are left out of analysis, and no finding is reported on them — not in the pull request comment, not as an inline comment, not in the check, and not in your hosted report.

| Pattern | Matches |
|---|---|
| `vendor/` or `vendor` | The directory `vendor` at any depth, and everything in it |
| `/dist` | Only the `dist` directory at the repository root |
| `*.snap` | Any file ending in `.snap`, at any depth |
| `src/generated/**` | Everything under `src/generated` at the repository root |
| `**/fixtures/**` | Everything under any directory named `fixtures` |
| `src/*.ts` | `.ts` files directly in `src`, not in its subdirectories |

The rules:

- Paths are relative to the repository root and use forward slashes. Matching is case-sensitive.
- `*` matches within one path segment. `?` matches one character. `**` matches any number of directories.
- A pattern with no slash matches at any depth. A pattern with a slash is anchored at the repository root.
- A pattern that matches a directory matches everything inside it.
- Negation (`!pattern`) is not supported. Brackets and braces are ordinary characters, not wildcards.
- At most 200 patterns, each at most 256 characters.

Two things `ignore` deliberately does not do:

- **It does not hide a problem in a file you did not ignore.** If a change in `src/api.ts` breaks something, that finding is reported even when the evidence for it points into an ignored file.
- **It does not change how the rest of the pull request is judged.** Ignored files are still read as context, so ignoring a path can only remove findings. It cannot cause a new one, and it does not change the threshold Faultline applies to the pull request.

### `instructions`

Free text, in plain language, that tells the AI review what your team considers intended. It is given to the review step that double-checks each candidate finding, next to the pull request's title and description.

What it can do: help that step recognize an intended change. A finding that describes exactly what your instructions say is intended can be shown as an advisory note instead of a likely regression.

What it cannot do:

- It cannot create a finding, raise a finding's severity, or make a finding count as confirmed.
- It cannot make a finding able to fail the check, and it does not change the threshold Faultline applies.
- It is not read by any of the checks that do not use AI. On a run without an Anthropic API key it has no effect.
- It is context, not a rule. The review step treats it the way it treats a pull request description, and it does not follow commands written in it.

Limits: 2,000 characters. Longer text is cut off at 2,000 and the run says so. The text is sent to Anthropic as part of the AI analysis, like your code and your pull request description; it is never uploaded to Faultline.

### `min_severity`

`low`, `medium`, or `high`. Findings below this severity are not posted on the pull request — neither in the comment nor as inline comments. The default is `low`, which posts everything.

- The comment says how many findings were left out.
- The findings still exist in the run's results and in your hosted report. This setting filters what is posted, not what is analyzed.
- **A regression that was confirmed by actually running tests is always posted**, whatever its severity. Those are the findings that can fail the check when `fail-on: execution` is set, and a failed check always has its explanation on the pull request.

## How this file relates to your other settings

Workflow inputs and your organization's settings keep precedence. `.faultline.yml` can only narrow what they already allow; it cannot turn anything on.

The order, from strongest to weakest:

1. **Organization limits** set in the Faultline portal (for example, inline comments not allowed).
2. **Workflow inputs** under `with:` in your workflow file.
3. **Organization defaults**, which apply only where the workflow says nothing.
4. **`.faultline.yml`**, which filters within whatever the three above produced.

| Setting | Decided by | What `.faultline.yml` can do |
|---|---|---|
| Whether inline comments are posted | `inline-comments` input, organization limit | Only reduce them, through `min_severity`. It cannot turn them on. |
| Whether findings can fail the check | `fail-on` input, organization default | Nothing through `min_severity`: it never hides a test-confirmed regression and never changes the check. Through `ignore`: a file that is left out cannot produce a finding, so it cannot fail the check. |
| What is uploaded to Faultline | `redaction` input, organization limit | Nothing. |
| Running your tests or generated tests | `differential-testing`, `repro-synthesis`, your plan | Nothing. It cannot enable either. |
| Sending the pull request title and description to the AI | `intent-context` input | Nothing. `instructions` is separate text from your own file; to send none, leave `instructions` out. |
| Findings your team has dismissed repeatedly | Your team's feedback | Both apply. A finding hidden by either one is hidden. |

## Mistakes in the file never fail your build

- **An unknown key** (for example a typo, `ignroe:`) is ignored, the rest of the file still applies, and the check summary carries a warning naming the key.
- **An invalid file** — YAML that does not parse, or a setting with a value that is not allowed, such as `min_severity: urgent` — is not applied at all. The run uses the defaults, exactly as if the file were absent, and says so: one line in the pull request comment, and the reason in the check summary. The file is never half-applied.
- **A file that cannot be read** is treated the same way.

In none of these cases does the check fail because of the file.

If the check summary says the Faultline engine version in use does not read `.faultline.yml` yet, your settings were not applied on that run. A later engine release will read it. Faultline chooses the engine release for each run, so there is nothing to change in your workflow.

## Who can change it

The file is part of your repository, and it is read from the pull request being reviewed. Anyone who can open a pull request can propose a change to it, exactly as they can propose a change to your workflow file. Two things keep that visible:

- The change is in the pull request's diff.
- Whenever the file changes what a comment shows, the comment says so: how many changed files were left out, and how many findings were not posted.

If Faultline's check is required for merging in your repository, protect the file the way you protect your workflow files — for example, add `.faultline.yml` to `CODEOWNERS`.

## Limits at a glance

| | Limit |
|---|---|
| File size | 64 KiB |
| `ignore` patterns | 200, each up to 256 characters |
| `instructions` | 2,000 characters |
| `min_severity` | `low`, `medium`, `high` |
| Keys | `ignore`, `instructions`, `min_severity` — nothing else |

YAML anchors and aliases are not supported.
