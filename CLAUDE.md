# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## What this is

Superpom is a Maven parent POM (`<packaging>pom</packaging>`) for Entur's routes and journey
planning Java/Spring Boot services: `org.entur.ror:superpom`, parented to
`spring-boot-starter-parent`, main branch `master`. There is no application source code, only
dependency management, plugin management and build config that child projects inherit.

`pom.xml` is the whole deliverable. Read it rather than trusting a summary here, and do not
copy its version numbers into this file where they will go stale.

## Build

```bash
./mvnw package                  # POM validation, enforcer, plugin wiring
./mvnw dependency-check:check   # OWASP scan, requires NVD_API_KEY
```

CI runs `./mvnw package -s .github/workflows/settings.xml`, but that settings file is not in the
repo: `push.yml` downloads it from `entur/ror-maven-settings` at build time. Locally, use plain
`./mvnw package`.

## What the POM manages

Two things in `dependencyManagement`: the Google Cloud BOM (`libraries-bom`, import scope) and
`logstash-logback-encoder`. Everything else comes from the Spring Boot parent.

The distinction between the two plugin blocks matters, and is the opposite of what "parent POM"
suggests:

- `build/plugins` is inherited by every child, but only the entries carrying `<executions>`
  actually run in its lifecycle: enforcer (Java version, `requirePluginVersions` at WARN),
  surefire, jacoco (agent, report, check), failsafe (`integration-test` + `verify`), source.
- Also in `build/plugins` but declared with no `<executions>`, so they run only when invoked
  explicitly: sonar (`./mvnw sonar:sonar`), javadoc, gitflow (`./mvnw gitflow:release-start`;
  `developmentBranch` = `master`, commits prefixed `[ci skip]`). Editing these three does not
  change any consumer's build.
- `pluginManagement` supplies config only when the plugin actually runs: compiler
  (`-Xlint:all`, forked, `-J-Xss4m`), surefire (`env=dev`), javadoc, release, jreleaser, OWASP
  dependency-check. For compiler and surefire that happens via default lifecycle bindings, so
  the config does reach children. OWASP has no `<execution>` declared anywhere, so despite the
  CVSS threshold being configured here the scan never runs by inheritance: it needs
  `./mvnw dependency-check:check`, or an execution added in the child. Adding a `<phase>` here
  will not help.

## Things that are easy to get wrong

**Every change ships to every consumer** (baba, uttu, lamassu, ...). Nothing here compiles
Java, so a green `./mvnw package` proves very little. Validate plugin and BOM bumps by forcing
real resolution, e.g. `./mvnw dependency:resolve-plugins` or `./mvnw <prefix>:help`, instead of
assuming a version that exists also works.

**`java.version` gates every JDK in the pipeline.** It feeds `maven-compiler-plugin`'s
`<release>`, the javadoc report in `<reporting>`, and the enforcer's `requireJavaVersion`, where
a bare value means that version *or higher* (`25` becomes `[25,)`). That floor binds consumers
too: adopting a new major means their local and CI JDKs must meet it, unless they set their own
`<java.version>`, which does override it cleanly and is the escape hatch to tell them about.

**Two JDKs need bumping, not one.** `java-version` in the `maven-package` job of
`.github/workflows/push.yml`, *and* the `java_version` input to `publish-release`, which
defaults to 21 in `entur/gha-maven-central`. Miss the second and the build goes green, then the
release dies at the enforcer during `./mvnw -Ppublication deploy`.

**Never test a Java bump with `-Djava.version=N`.** The name collides with the JVM's own
`java.version` system property, so `-D` sets the enforcer's requirement *and* the detected JDK
version to the same value: the rule passes vacuously and you get a false green. With the rule
hardcoded to `[25,26)`, `-Djava.version=99` reports "Detected JDK ... is version 99". Edit the
property in a scratch copy of the POM instead. Relatedly, `help:evaluate` reports the JVM's
value here, not the POM's.

**The released version is computed, not read from the POM.** A push to `master` runs
`entur/gha-maven-central` with `next_version: 'minor'`, which takes the POM's current version
and increments the *minor*, zeroing the patch. There is no manual release step. Hand-setting
`7.0.0-SNAPSHOT` therefore does not ship 7.0.0, it ships **7.1.0**; history confirms it, since
`6.0.0-SNAPSHOT` went out as `6.1.0` and Central has no 6.0.0 at all. To ship an exact version,
pass it for that run as `next_version: '7.0.0'`.

Commits containing `[skip ci]` are suppressed by GitHub's own native skip handling, not by the
`if:` guard in `push.yml`, which only matches the substring `ci skip`.

**`5.x` is a live maintenance line** for consumers still on Spring Boot 3.5 / Java 17, tracked
by Renovate and pinned to Spring Boot `<4.0.0` in `renovate.json`. Consumer-facing fixes may
need backporting. (`1.x` exists but has been dormant since 2023.)

**`dependencycheck-suppression.xml` is not in this repo.** The OWASP config names it as a
relative path, so it resolves in each child project. Adding it here will not apply downstream.

**The `sude` pin on `sonar-maven-plugin` is unproven, not verified-necessary.** The POM comment
explains the original `ClassNotFoundException` and its premises still hold, but the failure does
not reproduce here: sude 2.0.2 lands on the plugin classpath with *and* without the pin.
Superpom's own CI never runs Sonar, so it was only ever seen in a consumer. Reproduce it in a
consumer before removing the pin.

**`protobuf-java.version` and `grpc-java.version` must be re-checked on every
`google.cloud.bom-version` bump.** Spring Boot manages protobuf and grpc through *imported*
BOMs, which lose to superpom's imported `libraries-bom`, but it drives `protoc` and
`protoc-gen-grpc-java` from those two plain properties. If they drift from what libraries-bom
resolves, codegen and runtime diverge and proto classes throw
`ProtobufRuntimeVersionException: Detected incompatible Protobuf Gencode/Runtime versions`
on first load, *after* a completely green build. Renovate cannot catch this, since the
properties have no dependency attached. After bumping the BOM, confirm the pins still match:

```bash
./mvnw dependency:list        # in a child: what protobuf-java/grpc-core actually resolve to
./mvnw help:effective-pom     # in a child: grep <protoc> and compare
```

The general rule: Spring Boot's *direct* `dependencyManagement` entries win over superpom's
imported `libraries-bom`; its *nested BOM imports* lose to it. Overlapping artifacts where the
two disagree went from 1 to 36 when the parent moved to Spring Boot 4.1.

**Two publish targets coexist.** `distributionManagement` points at `entur2.jfrog.io`
Artifactory and `push.yml` exports `JFROG_*`, while the actual release path goes to Maven
Central through JReleaser. Know which one you are touching.

## Other

- Profile `publication` attaches sources and javadoc JARs and deploys to
  `target/staging-deploy` for JReleaser.
- Dependency updates come from Renovate (`renovate.json`), which auto-merges Spring Boot
  parent patch/pin/digest bumps.
