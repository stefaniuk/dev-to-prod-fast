# Dev to Prod Fast

Concise documentation for the automated release pipeline using Semantic Release, a GitHub App token, and GPG commit/tag signing.

## Overview

On every push to `main` the workflow (`.github/workflows/cicd-1-pull-request.yaml`) runs Semantic Release to:

1. Analyze commits (Conventional Commits) to decide the next version.
2. Write the new version into `VERSION`.
3. Commit the version file with a release message.
4. Create and (optionally) sign the tag with the imported GPG key.
5. Publish a GitHub Release (canonical release notes; no local changelog file maintained).

## Why a GitHub App Token (Not PAT)

- Least privilege: scopes and repo access constrained to the App installation.
- Ephemeral: short‑lived installation tokens reduce blast radius if leaked.
- Revocable: uninstalling or revoking the App immediately stops automation without touching user credentials.
- Auditability: actions taken with the App token are attributed to the App in GitHub’s audit logs.
- Separation: avoids broad user PATs that might cover many repos or scopes.

## Why the Signing Key Belongs to a Bot/User

GitHub only verifies GPG/SSH signatures against keys uploaded to a user (or machine/bot) account profile. Apps cannot host public keys. A dedicated bot account provides:

- Verified signatures ("Verified" badge) for release commits/tags.
- Clear authorship vs human developers (audit separation).
- Independent key rotation (rotate bot key without impacting developers).
- Reduced ambiguity: release provenance tied to a stable identity.

Use a bot-specific email (or GitHub noreply) that matches the key UID and the commit author email to ensure verification succeeds.

## Files

| File          | Purpose                                                    |
| ------------- | ---------------------------------------------------------- |
| `.releaserc`  | Plugin configuration (analyzer, notes, exec, git, github). |
| `VERSION`     | Plain text current version written during prepare phase.   |
| Workflow YAML | Implements release steps and key import.                   |

## Plugins & Flow

Order in `.releaserc`:

1. `@semantic-release/commit-analyzer`
2. `@semantic-release/release-notes-generator`
3. `@semantic-release/exec` writes `${nextRelease.version}` to `VERSION`.
4. `@semantic-release/git` commits `VERSION`.
5. `@semantic-release/github` publishes the GitHub Release (with notes only stored there).

## Conventional Commits Impact

| Type            | Example                               | Release bump |
| --------------- | ------------------------------------- | ------------ |
| feat            | `feat(ci): add exec plugin`           | minor        |
| fix             | `fix(ci): sign tag after release`     | patch        |
| perf            | `perf(core): optimize`                | patch        |
| docs            | `docs(readme): tidy`                  | no bump      |
| chore           | `chore(release): normalize changelog` | no bump      |
| refactor        | `refactor(ci): simplify script`       | patch        |
| BREAKING CHANGE | footer or `feat!:`                    | major        |

## GPG Signing

The workflow imports a GPG private key (bot/user) and sets global git config for commit signing. Tag signing is performed post-release by recreating the tag as signed (if that step is present). Ensure the public key is uploaded to the account matching `BOT_EMAIL` for “Verified”.

## Configuration

Variables:

- `GH_APP_ID`
- `GIT_SIGN_BOT_NAME`
- `GIT_SIGN_BOT_EMAIL`

Secrets:

- `GH_APP_PRIVATE_KEY` – PEM for the GitHub App.
- `GIT_SIGN_BOT_GPG_PRIVATE_KEY` – ASCII-armored GPG private key.
- `GIT_SIGN_BOT_GPG_PASSPHRASE` – (optional) passphrase; empty if key unprotected.

## Release Notes Strategy (No Local CHANGELOG.md)

We intentionally do NOT commit a `CHANGELOG.md` file. The GitHub Release notes are the single source of truth. This reduces churn and avoids merge conflicts common with a monolithic changelog file. Key rationale:

| Reason                 | Benefit                                                                            |
| ---------------------- | ---------------------------------------------------------------------------------- |
| Single source of truth | Prevents divergence between release page and repo file.                            |
| Smaller diffs          | Release commits touch only `VERSION` (clean history).                              |
| Conflict avoidance     | No manual merges of changelog edits across branches.                               |
| Performance            | Skips filesystem rewrite of a large markdown file each release.                    |
| API-friendly           | Consumers fetch structured notes via GitHub API without parsing markdown sections. |
| Auditable provenance   | Signed tag + release body form an immutable audit trail.                           |

If a materialized file is later required (e.g., packaging, offline docs), re-enable `@semantic-release/changelog` or generate an on-demand export pipeline.

## Adding a Feature

1. Create a branch and commit using `feat(scope): description`.
2. Open PR → merge to `main`.
3. Workflow computes next version (e.g. 1.2.0) and updates files.
4. Verify signed tag: check tag page for “Verified”.

## Troubleshooting

| Symptom                              | Cause                              | Fix                                             |
| ------------------------------------ | ---------------------------------- | ----------------------------------------------- |
| Tag creation fails with EDITOR unset | Forced tag signing without message | Use lightweight then re-sign (current approach) |
| VERSION not updated                  | Missing exec plugin                | Already fixed via `@semantic-release/exec`      |
| Changelog duplicates                 | Multiple initial sections          | Normalize (done)                                |

## Future Improvements

- Add commit lint action to enforce Conventional Commits pre-merge.
- Optionally publish artifacts (npm, container) using additional plugins.
- Add build metadata file (version + SHA + timestamp) if traceability needs increase.

---

This README focuses on practical usage; architectural scaling guidance removed for brevity. Restore from history if needed.
