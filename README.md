# Dev to Prod Fast

Concise documentation for the automated release pipeline using Semantic Release, a GitHub App token, and GPG commit/tag signing.

## Overview

On every push to `main` the workflow (`.github/workflows/cicd-1-pull-request.yaml`) runs Semantic Release to:

1. Analyze commits (Conventional Commits) to decide the next version.
2. Update `CHANGELOG.md` and write the new version into `VERSION`.
3. Commit those files with a release message.
4. Create and then re-sign the tag (if enabled) with the imported GPG key.
5. Publish a GitHub Release.

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

| File           | Purpose                                                               |
| -------------- | --------------------------------------------------------------------- |
| `.releaserc`   | Plugin configuration (analyzer, notes, changelog, exec, git, github). |
| `CHANGELOG.md` | Generated release notes (Keep a Changelog style).                     |
| `VERSION`      | Plain text current version written during prepare phase.              |
| Workflow YAML  | Implements release steps and key import.                              |

## Plugins & Flow

Order in `.releaserc`:

1. `@semantic-release/commit-analyzer`
2. `@semantic-release/release-notes-generator`
3. Two `@semantic-release/changelog` entries (title + standard) – can be consolidated.
4. `@semantic-release/exec` writes `${nextRelease.version}` to `VERSION`.
5. `@semantic-release/git` commits `VERSION` + `CHANGELOG.md`.
6. `@semantic-release/github` publishes release.

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

- Consolidate duplicate changelog plugin entry (keep only configured one).
- Add commit lint action to enforce Conventional Commits pre-merge.
- Optionally publish artifacts (npm, container) using additional plugins.

---

This README focuses on practical usage; architectural scaling guidance removed for brevity. Restore from history if needed.
