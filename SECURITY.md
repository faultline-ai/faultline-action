# Security policy

## Reporting a vulnerability

Email **security@tryfaultline.com**. Please do not open a public issue for a
suspected vulnerability.

Include what you can of:

- what you found and where (this Action, the Faultline GitHub App, the portal,
  or a Faultline service);
- the steps to reproduce it;
- the version you saw it on (the Action tag or commit SHA your workflow pins);
- what an attacker could do with it.

Do not include your source code, secrets or license token in the report. If we
need more detail we will ask for it.

## Scope

This repository holds the Faultline GitHub Action: `action.yml` and the bundled
`dist/index.js`. Reports about the Action, the engine it downloads, and
Faultline's services (`license.tryfaultline.com`, `api.tryfaultline.com`,
`app.tryfaultline.com`) are all welcome at the address above.

Please do not test against other customers' organizations or data, and do not
run tests that degrade the service for others.

## Supported versions

Fixes are released on the latest `v0.x.y` tag, and the floating `v0` tag moves
to it. If your workflow pins an older exact tag or a commit SHA, update the pin
to receive a fix.

## What the Action sends and to where

The README's [What leaves your runner](README.md#what-leaves-your-runner) and
[Network access](README.md#network-access) sections list what the Action sends
and the hosts it contacts.
