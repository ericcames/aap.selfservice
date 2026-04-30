# Security Policy

## Scope

This is a **demo repository** used to deploy the AAP Self-Service Automation Portal on the Red Hat Demo Platform (RHDP). It contains no production credentials, no customer data, and no live service endpoints. All sensitive values are resolved at runtime from environment variables and are never committed.

## Supported Versions

Only the latest commit on `main` is maintained.

## Reporting a Vulnerability

Because this is a demo repo with no production exposure, **open a public GitHub issue** to report any security concerns. There is no need to use private disclosure for this project.

When reporting, include:
- A description of the issue
- The file(s) affected
- Any suggested fix if you have one

## What Should Never Be Committed

As a reminder — these are never committed to this repo:

- Credentials, tokens, or passwords of any kind
- Inventory files under `inventories/rhdp-*/` (gitignored — `rhdp-sample-demo/` is the only exception)
- `docs/dev-environment.md` (gitignored)

If you spot any of the above committed by mistake, open an issue immediately.
