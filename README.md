# install-script-policy-allow-scripts

## Feature exercised

npm v11.16.0 `allowScripts` opt-in install-script policy (Phase 1):
the `allowScripts` field in `package.json` approves specific packages
to run install scripts; unapproved packages emit advisory warnings but
still install. Exercises new `allow-scripts` `.npmrc` key.

## Mend config

Bucket A — default-emit with `npm, node` pinned to `>=11 <12` and
`>=20 <21`. No additional dimensions: the policy is advisory-only in
Phase 1, so the resolved tree is lockfile-accurate regardless of
approval status.

The `.whitesource` file pins `npm: ">=11 <12"` so that Mend's
`install-tool` provisions npm 11.x (which introduced the
`allowScripts` field). Without this pin, an older npm version would
not understand the new field and the advisory-warning behavior could
not be reproduced.

## Expected dependency tree

- `esbuild` 0.25.4 — registry, group main, `hasInstallScript: true`,
  approved in `allowScripts`
- `@esbuild/linux-x64` 0.25.4 — registry, group optional (platform
  binary, installed on linux/x64 only)
- `husky` 9.1.7 — registry, group main, `hasInstallScript: true`,
  NOT listed in `allowScripts` (advisory warning expected at
  install time)
- `chalk` 5.4.1 — registry, group main, no install scripts

## npm v11.16.0 feature notes

- `allowScripts` block in `package.json` must NOT be mis-parsed as
  dependency scope data or as a phantom dependency set.
- Advisory warnings on stderr (for unlisted `husky`) must not cause
  the UA pre-step to fail or time out, resulting in an empty or
  partial dependency tree.
- All four packages must appear in Mend's output regardless of their
  approval status, because Phase 1 is advisory-only.
- `deny-scripts` entries (not present in this probe) must not be
  treated as absent packages.

## Resolver notes (UA NpmLockCollector)

Detection path: `package-lock.json` present → `NpmLockCollector` →
lockfileVersion 3 → `packages` block parsed, converted to v1-style
internally. No fallback to `NpmLsCollector` or `NpmFsCollector`
expected.

The `npm.runPreStep` toggle is relevant: if pre-step runs `npm
install`, npm 11.x will emit advisory warnings to stderr for `husky`.
The UA must not treat stderr warnings as fatal. If `npm.ignoreScripts`
is also set, esbuild's postinstall is suppressed, but the lockfile
still lists the package — it should still appear in the tree.

## Status

Untested — added from npm v11.16.0 release (2026-05-27).
Source: https://github.com/npm/cli/pull/9360
