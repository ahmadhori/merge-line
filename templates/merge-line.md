# merge-line profile

Copy this file to `.claude/merge-line.md` in your repository and fill in what
applies. Every section is optional: `/merge-line` falls back to the defaults
named in each one.

## Branch

The branch pull requests merge into. Default: the repository's default branch.

main

## Tests

How to run the tests of a few files, and the whole suite. Default: the
project's own test command, found in package.json, a Makefile or pyproject.

- Some files: `npm test -- <files>`
- Everything: `npm test`

## Generated files

Files that are regenerated after a merge, never merged by hand, and the command
for each. Default: lockfiles, with their package manager.

| File | Command |
|---|---|
| `package-lock.json` | `npm install` |

## Version

A command each PR runs before it is pushed, or "none". Default: none.

none

## Deploy groups

Which paths belong to which deploy, the one command that deploys it, and what
proves it is live. A group whose command is "publishes on merge" deploys
itself. Default: no groups; a merge is the end of the line.

| Group | Paths | Command | Live check |
|---|---|---|---|
| site | `web/**` | publishes on merge | `https://example.com` shows the new text |
| api | `api/**`, `shared/**` | `make deploy-api` | `https://api.example.com/health` returns the new version |

## Migrations

The folder whose new files make a PR "migration first", and the command that
applies them before the deploy. Default: none.

- Folder: `db/migrations/`
- Command: `make migrate`

## Review

How many PRs are reviewed ahead of the merge point, and with what.
Default: 4, with the `pr-reviewer` helper.

- Ahead: 4
- Reviewer: pr-reviewer
