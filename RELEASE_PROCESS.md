# Browser RUM Agent Release Process

This document describes how to release the browser RUM agent packages from this repository.

## Release Environments

| Environment | Description |
| --- | --- |
| CDN | Browser bundles published to CloudFront/S3 under `https://cdn.observability.splunkcloud.com/o11y-gdi-rum/`. |
| npmjs.com | npm packages published to the public npm registry. |
| GitHub Release | Release entry and downloadable artifacts published to `signalfx/splunk-otel-js-web`. |

For v3 and later, use the `cdn.observability.splunkcloud.com` CDN domain. The old `cdn.signalfx.com`
domain is legacy and `latest` is pinned to v2.5.x.

## Packages

| Package | CDN | npmjs.com | GitHub Release |
| --- | --- | --- | --- |
| `@splunk/otel-web` | Yes | Yes | Yes |
| `@splunk/otel-web-session-recorder` | Yes | Yes | Yes |
| `@splunk/rum-build-plugins` | No | Yes | Yes |

## Release Stages

| Version/tag example | CDN | npmjs.com | GitHub Release |
| --- | --- | --- | --- |
| `main` | Yes | No | No |
| `v3.1.0-alpha.1` | Yes | Yes, `alpha` dist-tag | No |
| `v3.1.0-rc.1` | Yes | Yes, `rc` dist-tag | No |
| `v3.1.0-beta.1` | Yes | Yes, `beta` dist-tag | Yes, prerelease |
| `v3.1.0` | Yes | Yes, default dist-tag | Yes |

The GitLab pipeline controls these stages:

- `main` publishes CDN artifacts only.
- `vX.Y.Z-alpha.N` and `vX.Y.Z-rc.N` publish to CDN and npm.
- `vX.Y.Z-beta.N` publishes to CDN, npm, and a GitHub prerelease.
- `vX.Y.Z` publishes to CDN, npm, and a GitHub release.

## CDN Version Paths

Tagged releases publish to multiple CDN paths.

For `v3.1.0`, production CDN paths are:

```text
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3.1.0/splunk-otel-web.js
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3.1/splunk-otel-web.js
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3/splunk-otel-web.js
```

And for session recorder:

```text
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3.1.0/splunk-otel-web-session-recorder.js
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3.1/splunk-otel-web-session-recorder.js
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3/splunk-otel-web-session-recorder.js
```

The exact version path is immutable. The minor and major paths are mutable convenience locks.

## Main Branch CDN Snapshots

Commits on `main` publish two CDN variants:

- Mutable `next`:

```text
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/next/splunk-otel-web.js
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/next/splunk-otel-web-session-recorder.js
```

- Immutable commit snapshot:

```text
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v<package-version>-<40-char-commit-sha>/splunk-otel-web.js
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v<package-version>-<40-char-commit-sha>/splunk-otel-web-session-recorder.js
```

Use the immutable snapshot for release validation when possible. It is safer than `next` because `next`
moves every time a newer `main` commit is deployed.

Example after a `3.1.0` release branch is merged to `main`:

```text
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3.1.0-<commit-sha>/splunk-otel-web.js
```

The agent also reports the full build identity when it is loaded through `next` or an immutable snapshot:

- `splunk.rumVersionFull`
- `splunk.rumVersionFullSessionRecorder`

These attributes can be used to confirm which exact CDN build produced telemetry.

## Prepare A v3.1.0 Release

1. Start from latest `main`.

```bash
git checkout main
git pull upstream main
git checkout -b chore/release-v3.1.0
```

2. Bump all package versions.

```bash
pnpm run version:bump 3.1.0
```

The version bump updates all workspace package versions and regenerates package `src/version.ts` files.
If the command fails, update the package versions manually, then run:

```bash
pnpm install
```

3. Update `CHANGELOG.md`.

Add a `3.1.0` section following the existing changelog format. Include:

- breaking changes, if any
- new features and improvements
- bug fixes
- dependency updates
- migration notes or customer-visible behavior changes

4. Run local checks.

```bash
pnpm run version:check
pnpm run build
pnpm run lint
pnpm run test:unit --browser.name chromium
```

Run focused integration tests if the release includes behavior that should be validated end to end:

```bash
pnpm run test:e2e
```

5. Commit and open the release PR.

```bash
git status
git add package.json pnpm-lock.yaml packages/*/package.json packages/*/src/version.ts CHANGELOG.md
git commit -m "chore(release): v3.1.0"
git push -u origin chore/release-v3.1.0
```

Request review from the BRUM team and wait for CI to pass before merging.

6. After merge, validate the main CDN snapshot.

Find the merge commit SHA:

```bash
git checkout main
git pull upstream main
git rev-parse HEAD
```

Then test the immutable snapshot URLs:

```text
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3.1.0-<commit-sha>/splunk-otel-web.js
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3.1.0-<commit-sha>/splunk-otel-web-session-recorder.js
```

Do not rely only on `next` for final validation, because `next` may point at a newer commit by the time
validation is complete.

7. Tag the release commit.

Create the tag on the merged release commit:

```bash
git tag v3.1.0
git push upstream v3.1.0
```

If you do not have permission to push tags, ask a BRUM maintainer to push the tag.

8. Monitor the GitLab release pipeline.

The tag pipeline should:

- build artifacts
- publish npm packages
- create a GitHub release
- publish CDN artifacts
- invalidate CloudFront for mutable paths

9. Validate production release artifacts.

Check npm:

```bash
npm view @splunk/otel-web version
npm view @splunk/otel-web-session-recorder version
npm view @splunk/rum-build-plugins version
```

Check CDN:

```text
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3.1.0/splunk-otel-web.js
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3.1.0/splunk-otel-web-session-recorder.js
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3.1/splunk-otel-web.js
https://cdn.observability.splunkcloud.com/o11y-gdi-rum/v3/splunk-otel-web.js
```

Check GitHub Releases for the `v3.1.0` release and verify the generated CDN snippets.

## Production Post-Release Updates

After a production release, update examples and integration test expectations that intentionally point at a
specific production version.

Create a follow-up branch:

```bash
git checkout main
git pull upstream main
git checkout -b chore/postrelease-v3.1.0
```

Update version references in:

- `packages/integration-tests/src/tests/cdn/index.spec.ts`
- `examples/`

Run relevant checks and open a follow-up PR.

## Troubleshooting

### Tag pipeline does not publish

Confirm the tag matches one of the supported formats:

```text
vX.Y.Z
vX.Y.Z-alpha.N
vX.Y.Z-beta.N
vX.Y.Z-rc.N
```

Also confirm the tag is on a commit where the root `package.json` version matches the tag without the `v`
prefix.

### `version:check` fails

The package version and generated `src/version.ts` file do not match for one or more packages. Run the
version bump again or update the generated version files with:

```bash
pnpm run version:bump 3.1.0
```

### CDN snapshot path is missing

The snapshot path is generated only by the `main` CDN release job. Branch pipelines do not publish commit
snapshot CDN artifacts.

### `splunk.rumVersionFull` is missing

The full version attribute is attached only when the agent is loaded from `next` or from an immutable locked
snapshot path matching:

```text
/o11y-gdi-rum/vX.Y.Z-<40-char-commit-sha>/
```

It is not expected for mutable major/minor/exact release paths such as `/v3/`, `/v3.1/`, or `/v3.1.0/`.
