# Proposal

## Why

A plugin is a jar that compiles against `plugin-api`. The module is built inside this repository and is not
published, so a plugin that lives in another repository cannot depend on it (#2261). The review of #2216 asked to
publish it with the Core release, under Core's version, so that a plugin author builds against exactly the version of
Core the plugin will run on.

## What Changes

- `plugin-api` is published to GitHub Packages (the Maven registry of this repository) as
  `com.epam.aidial:aidial-core-plugin-api`, by the existing release workflow. The Docker image, the tag and the GitHub
  release are produced as before.
- The module is renamed for Gradle in `settings.gradle` (`project(':plugin-api').name = 'aidial-core-plugin-api'`),
  so the project name is the artifact name. The folder stays `plugin-api`; the three `project(':plugin-api')`
  references in `server` and `plugin-samples` change. The Java package is unchanged.
- The module build file applies `java-library` and `maven-publish`. Jackson and Vert.x become `api` dependencies:
  their types (`Future`, `Buffer`, `ReadStream`, `ObjectNode`) are part of the contract, and today the published POM
  would list them as `runtime`, so a plugin author would not see them at compile time. The module also ships a
  sources jar, a Javadoc jar and the POM metadata Maven Central asks of every artifact.
- Versions: a `release-X.Y` build publishes `X.Y.0-rc.N`, a promotion publishes `X.Y.Z`, and a `development` build
  publishes `X.Y.0-SNAPSHOT` (the `-dev.N` part is replaced in the module's Gradle file, for the publication only).
  The CI version mechanism is unchanged.
- The shared workflow `epam/ai-dial-ci` gets an opt-in step for this (separate repository, separate change, see
  Impact). Core opts in from `.github/workflows/release.yml`.
- `docs/dynamic-settings/plugins.md` says how a plugin repository depends on the artifact; `CLAUDE.md` names the
  published artifact in the module table.

## Capabilities

No spec-level behaviour changes: this is build, release and documentation work, and Core serves and writes exactly
what it did. The change sets `skip_specs: true` in its `.openspec.yaml`. `plugin-registry` (how Core loads and runs
plugins) is not touched.

## Impact

- Motivation: #2261 "Plugins: publish the plugin-api module". This change covers its first two points (publish with
  Core's version from the release workflow; Vert.x and Jackson as `api`). It leaves out the third point, a stated
  compatibility rule: the docs keep saying that plugins are a preview and that a plugin jar is built for one Core
  version.
- Out of scope: Maven Central (a later change; the metadata, sources and Javadoc jars it needs are added now, the
  signing, the upload and the repository are not), removal of old rc and SNAPSHOT versions, a backport to
  `release-0.50`, and publishing any other module.
- Code: `settings.gradle`, `plugin-api/build.gradle`, `server/build.gradle`, `plugin-samples/build.gradle`,
  `.github/workflows/release.yml`, `docs/dynamic-settings/plugins.md`, `CLAUDE.md`.
- Dependencies between changes and PRs:
  - This branch is stacked on #2216 (`feat/plugin-system`), because `plugin-api` is not on `development` yet.
  - `epam/ai-dial-ci` must release the new opt-in input (a minor version, planned as 4.14.0) before the
    `release.yml` change of this branch can merge; pinning a tag that does not exist breaks the release workflow. The
    `release.yml` edit is therefore a commit of its own.
- Consumers of the artifact need a token even for a public package (a limit of the GitHub Packages Maven registry);
  the docs say so.
