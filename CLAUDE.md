# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Superpom is a Maven parent POM (`<packaging>pom</packaging>`) for Entur's routes and journey planning Java/Spring Boot microservices. It contains **no application source code** — only build configuration, dependency management, and plugin management that child projects inherit.

- **Group ID:** `org.entur.ror`
- **Parent:** `spring-boot-starter-parent` (Spring Boot 3.5.x)
- **Java version:** 17 (enforced by maven-enforcer-plugin)
- **Main branch:** `master`

## Build Commands

```bash
# Build (validate POM, run enforcer checks)
mvn package

# Build with Entur settings (as CI does)
mvn package -s .github/workflows/settings.xml

# OWASP dependency vulnerability check (requires NVD_API_KEY)
mvn dependency-check:check
```

## Key Configuration

### Dependency Management
- Google Cloud BOM (`libraries-bom`)
- Logstash Logback Encoder

### Plugin Management (inherited by child projects)
- **Compiler:** Java 17 with `-Xlint:all`
- **Surefire:** Sets `env=dev` system property for tests
- **Failsafe:** Integration test execution bound to `verify`
- **JaCoCo:** Code coverage reporting
- **OWASP Dependency-Check:** Fails build on CVSS >= 7; suppression file: `dependencycheck-suppression.xml`
- **JReleaser:** Publishes to Maven Central via Sonatype
- **Gitflow:** Development branch is `master`; commit messages prefixed with `[ci skip]`

### Profiles
- `publication` — Attaches sources and javadoc JARs, deploys to local staging directory for JReleaser

## Release Process

Releases are automated via GitHub Actions (`push.yml`). On push to `master`, the CI builds and then publishes to Maven Central using the reusable `entur/gha-maven-central` workflow with `next_version: 'minor'`. Version bumps in commit messages contain `[skip ci]` / `ci skip` to avoid recursive builds.

## Dependency Updates

Managed by Renovate (`renovate.json`). Spring Boot starter patch/digest updates are auto-merged.
