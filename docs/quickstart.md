# Faultline quickstart

Faultline reviews every pull request for regressions: changes that could break
behavior somewhere else in your codebase. The review runs inside your own GitHub
Actions and posts one comment and one check on the pull request.

This page covers only what is available today. Setup takes a few minutes and has
no sales call and no payment card.

- [Before you start](#before-you-start)
- [Set up](#set-up)
- [Read your first review](#read-your-first-review)
- [Give feedback](#give-feedback)
- [Add your Anthropic key (optional)](#add-your-anthropic-key-optional)
- [Repository configuration file](#repository-configuration-file)
- [Plans and allowances](#plans-and-allowances)
- [What is not available](#what-is-not-available)
- [Cancel or uninstall](#cancel-or-uninstall)
- [Troubleshooting](#troubleshooting)

## Before you start

You need:

- **A GitHub organization.** Faultline works with organizations, not personal
  accounts. If your repositories are under a personal account, create a free
  organization on GitHub and move them there first.
- **An organization owner.** Only an owner can install the GitHub App and accept
  the terms. Other members can sign in and see the portal read-only.
- **Code in TypeScript, JavaScript, or Python.** Other languages are not
  analyzed.
- **GitHub Actions** enabled on the repositories you want reviewed.

## Set up

Steps 2 and 3 can be done in either order.

1. **Sign in.** Open the [Faultline portal](https://app.tryfaultline.com) and
   sign in with GitHub. GitHub is the only sign-in method. Choose your
   organization from the list.

2. **Activate your organization.** On the **Setup** tab, under "Activate this
   organization", choose **Accept and activate**. This accepts the terms and
   puts your organization on the Free plan: 30 reviews a month, private
   repositories included, no payment card.

3. **Install the GitHub App and pick repositories.** On the **Setup** tab,
   choose **Install the GitHub App**. On GitHub's install screen, pick all
   repositories or only the ones you want reviewed. If you are not an owner,
   GitHub sends your request to an owner, and the step completes when they
   approve it.

4. **Merge the workflow pull request.** Faultline opens a pull request titled
   "Add Faultline PR regression analysis" in each repository you picked. It adds
   one file, `.github/workflows/faultline.yml`. Review it and merge it.
   Faultline never merges it for you.

   Merging before you activate is safe. Until your organization is activated,
   each run posts a neutral check titled "Faultline: not yet activated" and
   analyzes nothing. It never fails your build.

5. **Your license token arrives by itself.** Once the organization is activated
   and the App is installed, Faultline adds a secret named `FAULTLINE_TOKEN` to
   your organization's Actions secrets. There is nothing to copy or paste. The
   **License tokens** tab shows whether it has been delivered.

6. **Open a pull request.** Any pull request will do. The Faultline workflow
   runs and posts its review. The **Setup** tab marks setup complete when the
   first run arrives.

## Read your first review

**The comment.** Faultline posts exactly one comment per pull request and edits
it in place on every push. The first line is the verdict, and it is one of:

| First line | What it means |
|---|---|
| "N likely regressions" | Faultline found regressions and lists each one with the file and line it is based on. |
| "no regression found in the analyzed changes" | The full analysis ran and found nothing. |
| "deterministic checks passed — LLM analysis not run" | The run had no AI analysis (see [Add your Anthropic key](#add-your-anthropic-key-optional)). The checks that need no AI model found nothing. |
| "cannot certify this PR" | The analysis finished, but Faultline declines to vouch for the result on this change. Findings, if any, are still shown. Re-running does not change this. |
| "analysis incomplete" or "analysis failed" | Something stopped the run before it finished. Re-run the workflow. |

Below the verdict you get the findings, a "Test gaps" section, a "What to do
next" checklist, and a "Full report" link to a hosted copy of the report.

**The check.** A check named `Faultline` appears on the pull request.

- Green: the run completed. Findings are advisory, so the check can be green
  while the comment lists findings. Always read the comment.
- Neutral: the run was incomplete, could not certify the result, or the
  organization is not yet activated. Neutral is never a pass.
- Red: the run itself failed, or Faultline refused to run (for example, the
  license token is missing or revoked).

By default a finding never turns the check red. Whether a red check blocks
merging is decided by your own branch protection rules.

**Inline comments.** Up to five findings per pull request are also posted as
review comments on the exact lines they refer to. When Faultline has a fix that
fits inside the changed lines, the inline comment carries it as a GitHub
suggestion you can commit from the review screen. The suggestion is labeled
"Proposed fix (static — not execution-verified)": Faultline wrote it but did not
run it, so read it before you commit it. Faultline never commits anything
itself.

## Give feedback

Tell Faultline whether a finding was right by replying on the pull request. Put
the directive on its own line:

```
faultline up <code>
faultline down <code> false_positive
```

- `<code>` is the finding's short code. Every inline comment shows the exact
  line to copy, and all codes are listed under "Rate these findings" at the
  bottom of the main comment.
- `up` means the finding was a real catch. `down` means it was not.
- With `down` you can add one reason: `false_positive`, `wont_fix`,
  `not_actionable`, `duplicate`, or `style_disagreement`.
- Only organization owners, members, and repository collaborators can vote.
- Replies are read the next time Faultline runs on that pull request. An edited
  reply is not re-read; post a new one to change your vote.
- Only the directive is recorded. Anything else you write in the reply is
  ignored and is not sent to Faultline.

Finding types your team keeps voting down are moved into a collapsed section on
later pull requests. The **Learning** tab in the portal shows what your votes
have changed.

## Add your Anthropic key (optional)

Faultline's AI analysis runs on your own Anthropic API key. Without a key, every
run still completes: it uses only the checks that need no AI model, and the
comment says so. A missing key never fails a run.

To turn AI analysis on, create an API key in your Anthropic console and add it
to your GitHub organization's Actions secrets under the name
`ANTHROPIC_API_KEY` (organization **Settings → Secrets and variables →
Actions**). The workflow already reads that secret.

Faultline never sees, stores, or transmits this key. AI calls go from your
GitHub Actions runner straight to Anthropic, under your account and on your
Anthropic bill.

## Repository configuration file

A repository can add an optional `.faultline.yml` at its root to leave paths
out of the review, give the AI review context, and set the lowest severity
posted on the pull request. It needs Faultline Action v0.1.7 or later; a
workflow that uses `@v0` already has it, and on an older pinned version the
file is ignored. The settings are described in the
[repository configuration reference](repo-config.md).

## Plans and allowances

| Plan | Reviews per month |
|---|---|
| Free | 30 |
| Starter | 150 |
| Team | 1,500 |

- One review is one run of the Faultline workflow. Each push to a pull request
  starts a new run. Re-running the same workflow run does not count twice.
- The allowance resets on the first day of each calendar month (UTC).
- The **Plan** and **Billing** tabs show how many reviews you have used.

**When the allowance runs out, your build never fails.** Faultline keeps
reviewing every pull request with the checks that need no AI model, and the
comment states that the allowance was reached and the date AI analysis resumes.
This applies even when your own Anthropic key is set.

**Team only:**

- Cross-repository analysis across your organization.
- Backtesting: replaying Faultline against fixes already in your history.
- QA verification: running your tests and checking a preview deployment,
  including in a browser.

How to use these is described in the
[Faultline Action README](https://github.com/faultline-ai/faultline-action#readme).

**Changing plans.** An organization owner chooses Starter or Team on the
**Plan** tab and pays through Stripe's checkout page. Nothing is reinstalled. If
the Plan tab says paid plans are not available yet, stay on Free; the tab shows
them when they open. An organization that already has a subscription changes
it with **Manage billing** on the **Billing** tab instead.

## What is not available

- **Other code hosts.** Faultline works with GitHub only. There is no GitLab
  support.
- **Personal GitHub accounts.** Organizations only.
- **Chat with the bot.** Faultline does not answer questions or replies on the
  pull request. The only replies it reads are the `faultline up` and
  `faultline down` lines above.
- **Languages other than TypeScript, JavaScript, and Python.**
- **Execution verification by default.** Faultline can confirm a regression by
  running your tests, or a small test it writes, on your runner. This is off by
  default and is offered on the Team plan on request. It runs only when both
  are true: Faultline has enabled it for your organization, and you have turned
  it on in your workflow. The **Settings** tab shows whether it is enabled for
  your organization.

## Cancel or uninstall

**Cancel a paid plan.** On the **Billing** tab, an organization owner chooses
**Manage billing**, which opens Stripe's billing page, and cancels there.

- **Until the subscription ends, nothing changes.** You keep your paid plan.
  When the cancellation is scheduled for a later date, the Billing tab shows
  the date the plan ends, and an owner can resume the subscription in
  **Manage billing** before then.
- **When the subscription ends, your organization moves to the Free plan
  automatically.** Your license token stays the same and nothing is
  reinstalled, so the `Faultline` check does not turn red because you
  cancelled. Reviews continue on the Free allowance of 30 a month. If you have
  already used more than that this month, reviews continue with the checks
  that need no AI model until the allowance resets.
- **Team-only features stop.** A workflow that uses a Team-only mode
  (cross-repository indexing, backtesting, or QA verification) is refused and
  its run fails, with a message saying your plan does not authorize that
  mode. Remove those workflows, or set them back to the standard review.
- **To subscribe again,** an owner chooses Starter or Team on the **Plan**
  tab, as before.

The same happens when a subscription ends because the final payment failed.

If we set your organization up for you, rather than through the steps on this
page, the license token is revoked when the subscription ends and the
`Faultline` check fails with a message saying the subscription was revoked
or canceled. An owner can choose **Continue on Free** on the **Plan** tab to
carry on.

To stop altogether, uninstall as described next.

**Uninstall.** Remove the Faultline GitHub App from your organization's
**Settings → GitHub Apps**. When the App is removed, the license token it
delivered is revoked. Two things stay behind in your organization, and you
should remove them yourself:

- `.github/workflows/faultline.yml` in each repository. If you leave it, the
  `Faultline` check fails on every pull request.
- The `FAULTLINE_TOKEN` organization secret (and `ANTHROPIC_API_KEY`, if you
  added it only for Faultline).

Uninstalling does not cancel a paid subscription. Cancel it on the **Billing**
tab first.

## Troubleshooting

**The workflow pull request did not appear.**

- Check that the App is installed on that repository. If you chose "only select
  repositories", add the repository on the App's settings page on GitHub.
- If you are not an organization owner, your install is a request. Nothing is
  opened until an owner approves it.
- If the repository already has `.github/workflows/faultline.yml` on its default
  branch, Faultline treats it as set up and opens nothing.
- If you closed the pull request without merging, Faultline does not reopen it
  by itself. Reinstall the App to get a new one, or copy the workflow from the
  "Add the workflow" step on the **Setup** tab and commit it by hand.
- The **Setup** tab lists the state of the setup pull request for each
  repository.

**The check says the token is missing, or "not yet activated".**

- "Not yet activated" (neutral): the organization has not accepted the terms.
  Finish step 2 above.
- "No FAULTLINE_TOKEN was presented" (red): open the **License tokens** tab. If
  the token is not delivered, an organization owner can choose **Deliver
  token**. If delivery keeps failing, check that the App is still installed,
  then mint a token on that tab and add it by hand as an organization Actions
  secret named `FAULTLINE_TOKEN`.
- If the token is delivered but one repository still fails, check on GitHub
  that the `FAULTLINE_TOKEN` organization secret is visible to that repository.
  Adding the repository to the App's repository list updates this for you.

**The comment says "analysis incomplete".**

A step of the run did not finish, and the comment names which one. The findings
shown cover only the steps that did run, and the check is neutral. Re-run the
workflow from the pull request's Checks tab. If it keeps happening, send us the
link to the run.

**The comment says AI analysis was not run.**

Either no `ANTHROPIC_API_KEY` secret is set (or its name is misspelled), or the
month's allowance is used up. The comment says which.

**The check fails saying your plan does not authorize a mode.**

The workflow asks for a mode that is Team only (see
[Plans and allowances](#plans-and-allowances)) and your organization is on
Free or Starter. This also happens after a Team subscription ends. Remove the
`mode` line from the workflow to get the standard review, or choose Team on
the **Plan** tab.

**Which languages are supported?**

TypeScript, JavaScript, and Python. Files in other languages are skipped.

**Something else?**

Email [support@tryfaultline.com](mailto:support@tryfaultline.com).
