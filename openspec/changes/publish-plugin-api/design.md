# Design

## Context

See proposal.md for why. What exists today:

- `.github/workflows/release.yml:19` calls the shared workflow `epam/ai-dial-ci/.github/workflows/java_release.yml@4.13.2`
  with `secrets: inherit`. Its single `release` job runs, in this order: `Set version` (a `sed` on the root
  `build.gradle`, `java_release.yml:220-223`), `build_docker` (`:224`), `publish_tag_release` (`:248`, only on
  `release-*`). JDK and Gradle are already set up by `java_prepare` (`:216`), the job has `packages: write`
  (`:174-177`), and the job's `GITHUB_TOKEN` shows `Packages: write` in a real run log. The workflow has no
  `workflow_call` outputs and the `calculate_version` job exposes only `next-version`, `is-latest`,
  `latest-stable-tag` and `changelog-mode` (`:156-163`).
- The version comes from `semantic_versioning`: `X.Y.0-dev.N` on `development`, `X.Y.0-rc.N` on `release-X.Y`,
  `X.Y.Z` on promotion (`ai-dial-ci/actions/semantic_versioning/action.yml`). The root `build.gradle:8-10` sets
  `group = 'com.epam.aidial'` and `version = "0.0.0"` for every module in `allprojects`, which is the line the
  `sed` rewrites, so `plugin-api` already gets the release version.
- `plugin-api/build.gradle` declares Jackson and Vert.x with `implementation`. No module applies `maven-publish`.
  `server/build.gradle:7` and `plugin-samples/build.gradle:4,8` refer to `project(':plugin-api')`;
  `settings.gradle:13` includes it.
- The Docker build runs Gradle inside the image (`Dockerfile`), so the runner holds no jars to publish; the publish
  step has to run Gradle itself.
- Precedent in `ai-dial-ci`: `node_release.yml` has a `publish-enabled` input and publishes between `build_docker` and
  `publish_tag_release`, skipping a version that already exists (`npm view`); the Python workflows use
  `--skip-existing`. In this organisation `epam/ai-dial-keycloak-helpers` publishes a Gradle library to GitHub
  Packages with `maven-publish` and `GITHUB_TOKEN` (job permission `packages: write`; its 401 was fixed by adding
  that permission).

## Goals / Non-Goals

**Goals:** a plugin repository can `compileOnly` the artifact at the version of the Core it targets; the release
workflow publishes it with no extra manual step; nothing in the Docker, tag and release flow changes; the build file
is ready for Maven Central without a restructure.

**Non-Goals:** Maven Central itself (signing, a Central repository, the upload), removal of old versions, publishing
any other module, a compatibility rule for plugins, any change to what Core serves or writes.

## Decisions

### D1. The publish step lives in the shared `java_release.yml`, opt-in

A step after `build_docker` and before `publish_tag_release`, guarded by a new input `publish-enabled` (boolean,
default `false`, so the other Java repositories are unaffected), like `node_release.yml`. A second input,
`publish-package-names` (newline list of `group.artifactId`), is used only by the existence check (D6). The step runs
`./gradlew publish` with `GPR_USERNAME=${{ github.actor }}` and `GPR_PASSWORD=${{ secrets.GITHUB_TOKEN }}`, the names
the root `build.gradle:27-36` already reads. Publishing after `build_docker` means a failed Trivy scan or image build
publishes nothing; publishing before the tag means a tag implies a published artifact.

Rejected:
- A second job in Core's `release.yml` that needs `release`: the reusable workflow exposes no version, and
  recomputing it after the tag exists gives the next version (`semantic_versioning` bumps from the tag it finds).
- A separate workflow on `release: created` (the GitHub tutorial's trigger): `publish_tag_release` creates the
  release with `ncipollo/release-action`, whose token defaults to `github.token`, and events caused by `GITHUB_TOKEN`
  do not start workflows. A `push: tags` trigger would fire (the tag is pushed with the robot token), but it has no
  `development` builds and runs apart from the Docker result.
- Manual `workflow_dispatch` with a version input (keycloak-helpers): skips the automatic versioning.

### D2. Artifact `com.epam.aidial:aidial-core-plugin-api`, named in `settings.gradle`

`project(':plugin-api').name = 'aidial-core-plugin-api'` after `include 'plugin-api'`. Gradle then derives the
artifactId, the jar name and the project path from that name, so nothing in the build file can drift from it. A
module cannot rename itself: `name` is read-only in a build file (checked on Gradle 8.10.2, "Cannot set the value of
read-only property 'name'"). The folder, the sources and the Java package `com.epam.aidial.core.plugin` stay. The
three `project(':plugin-api')` references become `project(':aidial-core-plugin-api')`. The server distribution's
`lib/` jar is renamed with it; no file under `server/src`, `Dockerfile` or `docker-entrypoint.sh` names the jar.

Rejected: an `artifactId` override on the publication (works from the module's own file with zero ripple, but hides
the name in the publishing block and leaves the jar named `plugin-api-*.jar`); renaming the folder (the same
result with more churn, and a conflict with every edit under `plugin-api/` in #2216); the default `plugin-api` (too
generic: every DIAL Gradle repository shares the group `com.epam.aidial`, so the artifactId is the only
disambiguator; the one artifactId published in the organisation today is `aidial-keycloak-mappers`). GitHub accepts
lowercase letters, digits and hyphens in an artifactId; the name complies. The coordinates are permanent once
published.

### D3. Jackson and Vert.x are `api` dependencies

`plugin-api/build.gradle` applies `java-library` and declares `jackson-databind` and `vertx-core` with `api`. On a
scratch copy of this build (Gradle 8.10.2) `implementation` produced `<scope>runtime</scope>` in the POM and no
dependencies in the `apiElements` variant of the Gradle module metadata; `api` produced `compile` and listed them.
The plain `java` plugin has no `api()` (the build fails with "Could not find method api()"), hence `java-library`.
This is the second point of #2261. `server` and `plugin-samples` are unaffected: `server` declares both libraries
itself and `plugin-samples` declares them `compileOnly`.

### D4. `-dev.N` becomes `-SNAPSHOT` in the module's Gradle file

The publication version is `project.version` with a trailing `-dev.N` replaced by `-SNAPSHOT`; any other version is
unchanged. A `development` push thus overwrites one rolling coordinate, the Maven counterpart of the rolling
`development` image tag (`java_release.yml:234`). The CI version mechanism and the version file the server reports
(`server/build.gradle`, `distributions`) keep `-dev.N`. Maven orders `0.51.0-dev.5` above `0.51.0` (measured with
`maven-artifact` 3.9.9 `ComparableVersion`; Gradle puts `dev` lowest), while `-rc.N` < `-SNAPSHOT` < release in both,
so a SNAPSHOT never outranks a release.

Rejected: publishing `-dev.N` (immutable and pinnable, but mis-ordered in Maven and one permanent version per push,
about five a day); publishing nothing from `development` (a plugin author could not build against the `development`
Core); computing the version in the shared workflow and passing it to Gradle (the repository decided the CI version
mechanism stays as is).

### D5. Credentials: `GITHUB_TOKEN` only

The Maven registry is repository-scoped, so the workflow's token may publish packages of the workflow's repository
(GitHub Packages permissions documentation); no package-level grant is needed, unlike GHCR. `github.actor` is only the
Basic-auth user name. A robot token would also publish, but it widens exposure and its publishes raise
`registry_package` events, which `deploy-development.yml` listens to.

### D6. An existing rc or stable version is skipped, anything else fails

On non-`development` branches the step first lists the package's versions
(`GET /orgs/{owner}/packages/maven/{package}/versions`) and skips the publish when `next-version` is there, as
`node_release.yml` does with `npm view`. The tag is created after the publish, so a rerun after a cancelled run
(the workflow uses `cancel-in-progress`) computes the same version. GitHub Packages answers a second upload of a
non-SNAPSHOT version with `409 Conflict` (community reports only; GitHub's documentation does not say). A SNAPSHOT is
always republished. Any other failure fails the run, because Gradle stops at the first rejected file and a partial
version has to be deleted by hand.

Rejected: treating a `409` as success (can hide a partial upload and a spurious `409`); no handling (the keycloak
choice; a rerun then needs manual cleanup).

### D7. Foundation for Maven Central, nothing more

Added now because Central requires them of every artifact: full POM metadata (name, description, url
`https://github.com/epam/ai-dial-core`, Apache-2.0, scm, developers), a sources jar and a Javadoc jar
(`:plugin-api:javadoc` builds; the module has no Lombok). Not added: signing, a Central repository, the
`ossrh-staging-api` upload step. `epam/gflog` (commit 203e8e5) is the reference for the later change: `maven-publish`,
`signing` with in-memory keys, and a small Gradle task that performs the staging-API steps. Central would receive
only rc and stable versions (releases there are immutable; SNAPSHOTs need a per-namespace switch).

### D8. `calculate_version` forwards `is-release` and `is-release-candidate`

`semantic_versioning` already computes both (`action.yml`); `node_release.yml` forwards them. They are added to the
job outputs now, with a `# TODO: huh` comment, for the Central gating of D7. Unused until then.

## Effect per request

None. Core's request path, passes over a body, Redis and blob round trips are untouched; this is build, release and
documentation work. Not measured, nothing to measure. The only runtime-visible difference is the jar name in the
server distribution's `lib/`.

## Risks / Trade-offs

- [The `409` behaviour and the existence check rest on community reports and on `GITHUB_TOKEN` being able to list
  package versions] → the fork dry run (tasks.md) runs both; D6 fails loudly if the check is wrong.
- [GitHub Packages may reject the `-rc.N` form] → the same dry run; GitHub's documentation restricts only the owner
  and the artifactId.
- [A cancelled run can leave a partial rc upload] → delete the version in the package settings, rerun.
- [Anyone reading the package needs a token, even for a public package (documented limit of the registry)] → the docs
  say so; Maven Central is the later answer for plugin authors outside the organisation.
- [The root build excludes `netty-transport-native-epoll` from every configuration, and the exclusion lands in the
  published POM and module metadata] → accepted; it only affects a consumer's own transport choice.
- [A SNAPSHOT cannot be pinned] → intended for a rolling channel; rc and stable versions are immutable.
- [Each SNAPSHOT upload adds a set of timestamped files under the one `X.Y.0-SNAPSHOT` version plus an updated
  `maven-metadata.xml` (seen when publishing to a local file repository); whether GitHub Packages keeps the old
  timestamped files, and how fast they pile up at about five `development` pushes a day, is not checked] → look at the
  package page after the fork dry run; the clean-up is a follow-up.
- [`:aidial-core-plugin-api:javadoc` prints 12 "no @param / no @return / no comment" warnings on the existing API, and
  the Javadoc jar ships them] → not part of this change; they are warnings, the build passes.
- [The directory and the Gradle project name differ] → a comment on the `settings.gradle` line.

## Migration Plan

1. Release `epam/ai-dial-ci` with the opt-in input (planned 4.14.0).
2. Merge the Core change without the `release.yml` commit; nothing publishes yet.
3. Merge the `release.yml` commit (pin the new tag, `publish-enabled: true`,
   `publish-package-names: com.epam.aidial.aidial-core-plugin-api`) only after step 1.
4. The first `development` push publishes a SNAPSHOT; the first `release-*` push publishes an rc.

Rollback: set `publish-enabled: false` or revert the pin; published versions can be deleted from the package page.
No data, setting or API migrates.

## Open Questions

- Whether `GITHUB_TOKEN` may delete a version it published, for a future cleanup of old rc and SNAPSHOT versions.
