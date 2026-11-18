---
name: github-release
description: Publish a verified downstream release of UnitVectorY-Labs/delete-package-versions, including its TypeScript bundle and vX.Y.Z-uvy.N version convention. Use when asked to cut, tag, or publish a release of this repository.
---

# GitHub release

Publish the compiled TypeScript action from an exact, tested commit on `main`.
This skill is adapted from the [kuberollouttrigger-action release skill](https://github.com/UnitVectorY-Labs/kuberollouttrigger-action/blob/main/.agents/skills/github-release/SKILL.md), with this repository's build and downstream versioning conventions.

## Choose the downstream version

Inspect releases and remote tags before choosing a version:

```sh
gh release list --repo UnitVectorY-Labs/delete-package-versions --limit 20
git fetch origin --tags
git tag --list '*-uvy.*' --sort=-version:refname
```

The first fork release was `v5.0.1-uvy.1`. Its `5.0.1` base preserves the inherited upstream 5.x interface, while `uvy.N` identifies this fork's maintenance revisions. For another maintenance release on that base, increment only `N`: `v5.0.1-uvy.1` → `v5.0.1-uvy.2` → `v5.0.1-uvy.3`. Compute the next unused revision from current releases and tags; the example is not a permanent target. If a user supplies a version, validate it against this scheme. An intentional upstream rebase or compatibility change needs an explicitly chosen new base.

The private npm package's `0.0.0` version is an inherited build placeholder, **not the release version**. Do not run `npm version` or change package versions just to cut a release. Although the suffix has SemVer prerelease syntax, this fork publishes these as ordinary stable GitHub releases (`--prerelease=false`). Do not replace upstream tags or create/move `v5`, `v5.0`, or another moving alias as part of a maintenance release.

## Validate the release commit

Honor any authorization already supplied for merging and releasing. Preparation can proceed before publication is authorized; only publish when it is within the user's request.

1. Ensure source, configuration, dependency lockfile, license records, and generated bundle changes have been reviewed in a PR. Wait for all applicable PR checks to pass, then merge when authorized.
2. Fetch and update local `main` with `git pull --ff-only`. Confirm a clean working tree and that `HEAD` equals `origin/main`. Record the full release SHA. Verify the checks for that exact main commit, including `Package and Smoke Test`, `Check dist/`, `Licensed`, and the security scan. A deliberately gated integration dispatch job may be skipped in this fork; report that accurately.
3. Use Node.js 24, matching `action.yml` and CI. Build from a clean dependency install:

   ```sh
   npm ci
   npm run pack
   git diff --exit-code -- dist/
   ```

   `pack` runs formatting, lint, Jest tests, and TypeScript compilation, then bundles `lib/main.js` with `ncc`. Consumers execute the checked-in `dist/index.js` through `action.yml`; compiling only `lib/` is insufficient. Confirm a second `npm run pack` produces the same tracked `dist/` files and leaves the worktree clean. If validation changes tracked files, put the correction through the PR/check process before tagging.
4. Confirm the CI license check passes. Dependency changes may need reviewed `.licenses/` updates using the same Licensed version/configuration as `.github/workflows/licensed.yml`.
5. Confirm the bundled zero-deletion smoke test passes with `num-old-versions-to-delete: 0`, as in `.github/workflows/test.yml`. Do not exercise actual deletion merely to release a dependency update. Real package deletion requires a separately authorized target and scope.

If checks fail, fix the concrete failure and rerun the affected checks before publication. Do not treat a rerun of an unchanged dependency conflict as a fix.

## Publish the exact verified commit

Use the selected full tag and recorded SHA below. Recheck the remote tag does not already exist before creating it. A release retry must inspect existing tag/release state first; never force-update a published tag.

```sh
# Example values: derive these from the release inspection and verified main.
release_tag=v5.0.1-uvy.2
release_sha=$(git rev-parse HEAD)
git tag "$release_tag" "$release_sha"
git push origin "refs/tags/$release_tag"
```

Write release notes to a temporary Markdown file. Describe the user-visible changes since the previous fork release, link the merged PR and comparison, summarize actual validation, and include the full commit pin:

```yaml
uses: UnitVectorY-Labs/delete-package-versions@<full-release-sha> # v5.0.1-uvy.N
```

Publish with the reviewed notes and existing remote tag:

```sh
gh release create "$release_tag" \
  --repo UnitVectorY-Labs/delete-package-versions \
  --verify-tag --target "$release_sha" \
  --title "$release_tag — UnitVectorY-Labs maintenance release" \
  --notes-file /path/to/release-notes.md \
  --prerelease=false --latest
```

This is a GitHub Action release, not an npm publication. The inherited `publish-immutable-action.yml` workflow is optional and gated for upstream or `IMMUTABLE_ACTION_PUBLISHING_ENABLED=true`; ordinary fork releases use Git tags and GitHub releases. Do not enable optional OCI publishing or change its gate just to release.

## Verify publication

```sh
git ls-remote origin "refs/tags/$release_tag"
gh release view "$release_tag" \
  --repo UnitVectorY-Labs/delete-package-versions \
  --json url,tagName,isDraft,isPrerelease,targetCommitish
gh run list --repo UnitVectorY-Labs/delete-package-versions --commit "$release_sha" --limit 30
```

Confirm the remote lightweight tag resolves to the verified SHA, the release is published and not marked prerelease, and applicable post-release jobs succeeded. A skipped optional publishing job is expected when its gate is disabled. Report the release URL, full SHA, merged PR, and validation outcome. If publication partially succeeds, report its actual state and repair only within the existing release authorization; never delete or replace a published version to retry.
