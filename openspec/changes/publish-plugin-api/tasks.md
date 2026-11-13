# Tasks

Run Gradle with a JDK 21 as `JAVA_HOME` (or let the toolchain resolver fetch one). Groups 1 and 2 are Core, group 3 is
the `epam/ai-dial-ci` repository (its own PR), group 4 is the last Core commit.

## 1. Gradle: the module name and the publication

- [x] 1.1 `settings.gradle`: after `include 'plugin-api'` add `project(':plugin-api').name = 'aidial-core-plugin-api'`
  with a comment that the folder keeps its name; change the three `project(':plugin-api')` references in
  `server/build.gradle` and `plugin-samples/build.gradle` (design D2). Verify `./gradlew projects` lists
  `:aidial-core-plugin-api` and no `:plugin-api` (done), and `./gradlew :aidial-core-plugin-api:compileJava
  :plugin-samples:compileJava :plugin-samples:test` passes (done). `:server:compileJava` and `:storage:compileJava` need the
  jclouds package, so a token with `read:packages` (deferred: the `gh` token here lacks it; every project, `:server`
  included, configures under the new name).
- [x] 1.2 `plugin-api/build.gradle`: apply `java-library` and `maven-publish`; `api` for `jackson-databind` and
  `vertx-core`; `java { withSourcesJar(); withJavadocJar() }`; one `MavenPublication` from `components.java` with the
  POM metadata (name, description, url, Apache-2.0, scm, developers) (design D3, D7). Verify
  `./gradlew :aidial-core-plugin-api:generatePomFileForMavenPublication :aidial-core-plugin-api:generateMetadataFileForMavenPublication`
  gives `<artifactId>aidial-core-plugin-api</artifactId>`, both libraries in `compile` scope, and dependencies in the
  `apiElements` variant of `module.json`.
- [x] 1.3 Same file: the publication version replaces a trailing `-dev.N` with `-SNAPSHOT` and keeps any other version
  (design D4). Verify in a scratch copy of the repository (not committed): after the CI `sed` sets the root version to
  `1.2.0-dev.7`, `1.2.0-rc.1` and `1.2.0` in turn, the generated POM and the files published to a local file
  repository carry `1.2.0-SNAPSHOT`, `1.2.0-rc.1` and `1.2.0`, with the jar, sources jar, Javadoc jar, POM and module
  metadata present, and the remote file names follow the publication version.
- [x] 1.4 Same file: the `GitHubPackages` repository `https://maven.pkg.github.com/epam/ai-dial-core`, credentials
  read as in the root `build.gradle:27-36` (`gpr.user` or `GPR_USERNAME`, `gpr.key` or `GPR_PASSWORD`) (design D1).
  Verify `./gradlew publish --dry-run` runs only `:aidial-core-plugin-api:publish` and its
  `publishMavenPublicationToGitHubPackagesRepository`, no other module.
- [ ] 1.5 `./gradlew checkstyleMain checkstyleTest`, and the touched test classes: `./gradlew :server:test --tests
  "com.epam.aidial.core.server.PluginManagerTest" --tests "com.epam.aidial.core.server.EmbeddedOriginApiTest"`.
  Done: checkstyle and `:plugin-samples:test` on `:aidial-core-plugin-api` and `:plugin-samples`. Left: checkstyle on the
  other modules and the two `:server` test classes (deferred: need a token with `read:packages` for the jclouds package;
  the change touches no Java source and only the project path in `server/build.gradle`).

## 2. Docs

- [x] 2.1 `docs/dynamic-settings/plugins.md`: how a plugin repository depends on the artifact (the repository URL, a
  token with `read:packages` even for a public package, `compileOnly`, JDK 21, what `-SNAPSHOT`, `-rc.N` and a plain
  version mean); no compatibility rule. Verify the Gradle snippet against a scratch project that resolves the artifact
  from a local file repository from task 1.3.
- [x] 2.2 `CLAUDE.md`: the `plugin-api` row of the module table names the published artifact
  `com.epam.aidial:aidial-core-plugin-api` and the folder/project name difference. Verify the table still lists every
  module in `settings.gradle`.

## 3. `epam/ai-dial-ci` (separate repository and PR)

- [x] 3.1 `.github/workflows/java_release.yml`: input `publish-enabled` (boolean, default `false`); the publish step
  between `build_docker` and `publish_tag_release` runs `./gradlew publish` with `GPR_USERNAME` and `GPR_PASSWORD` from
  `github.actor` and `GITHUB_TOKEN`; no existence check, a second upload of an rc or stable version fails the run
  (design D1, D5, D6). Verify the file parses as YAML (done).
- [x] 3.2 Same file: `is-release` and `is-release-candidate` added to the `calculate_version` outputs with a
  `# TODO: huh` comment (design D8). Verify the output names match `actions/semantic_versioning/action.yml`.
- [x] 3.3 `README.md` "Java (gradle)": document the input, the `maven-publish` requirement, the credential names and
  the rerun behaviour. Verify the documented snippet matches the input name of 3.1 (done; no mention of the removed
  `publish-package-names` is left).
- [x] 3.4 The workflow change is in the fork `nepalevov/ci` (checkout `C:\Projects\GitHub\nepalevov\ci`): a
  `chore: sync with upstream` commit and a `feat:` commit, self-references set with its `fork.sh` helper, pushed to
  its `main`, which is the test ref of the polygon below.
- [ ] 3.5 The `epam/ai-dial-ci` PR: self-dependencies updated with `update-self-dependencies.sh`, then push and open
  the PR (deferred: needs the user's go-ahead to push to the `epam` repository).

## 4. Core release workflow (last commit, merges after ai-dial-ci is released)

- [ ] 4.1 `.github/workflows/release.yml`: pin the new `ai-dial-ci` tag (planned 4.14.0), add
  `publish-enabled: true`. Verify the diff is these lines only, in a commit of its own (deferred: the tag does not
  exist until ai-dial-ci is released). The polygon branch carries its own variant of this edit, pointing at
  `nepalevov/ci@main`; it is never merged into `feat/issue-2261`.

## 5. Integration checks

- [ ] 5.1 Dry run in the fork `nepalevov/ai-dial-core` (checkout `C:\Projects\GitHub\nepalevov\ai-dial-core`, branch
  `polygon/publish-plugin-api`, which calls `nepalevov/ci@main`): a `development` push publishes `X.Y.0-SNAPSHOT` and a
  second push overwrites it; a `release-X.Y` push publishes `X.Y.0-rc.0`, and a rerun fails on the `409` as designed;
  the Docker, tag and release steps still pass; a scratch project resolves the artifact with a read-only token. The
  fork, the polygon branch, the `ACTIONS_BOT_TOKEN` secret and the ci test ref are ready; no run has been started
  (deferred: waits for the user to trigger it, it publishes to the fork's packages).
- [ ] 5.2 After merge: the first `development` push of Core shows a SNAPSHOT under the repository's packages, the first
  `release-*` push an rc (deferred: needs the merge and the ai-dial-ci release).

## Workflow follow-up

- Write `review.md` in the change folder, then `npx -y @fission-ai/openspec@1.14.3 archive publish-plugin-api -y` and
  `npx -y @fission-ai/openspec@1.14.3 validate --specs --strict`.
- `npx -y @fission-ai/openspec@1.14.3 validate --all --strict` before each commit.
