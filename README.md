# maven-semantic-release-sample

A sample project that validates **automated, Conventional Commits-based releases** for a Java/Maven project.

As a modern (2026) setup, it uses Google's [release-please](https://github.com/googleapis/release-please) (`maven` strategy) to automate version management and releases.
(Previous setup: `@conveyal/maven-semantic-release` + `semantic-release@15` (npm) / JDK 8 — removed, since OSSRH was shut down and `nexus-staging-maven-plugin` is deprecated.)

## Release flow

1. Push commits to `master` using **Conventional Commits**:
   - `fix:` → patch, `feat:` → minor, `feat!:` / `BREAKING CHANGE:` → major
2. release-please automatically opens/updates a **Release PR** containing the CHANGELOG and the `pom.xml` version bump.
3. **Merging** the Release PR creates the `vX.Y.Z` tag and a GitHub Release, and the `publish` job builds the jar with **JDK 11** and attaches it to the Release.
4. release-please then opens a follow-up PR that bumps to the next `-SNAPSHOT` version.

## Files

| File | Purpose |
| --- | --- |
| `release-please-config.json` | release-please config (`release-type: maven`) |
| `.release-please-manifest.json` | Current released version (single source of truth) |
| `.github/workflows/release.yml` | Runs release-please + builds/attaches the jar |
| `pom.xml` | `maven.compiler.release=11` to guarantee a JDK 11 build |

## Required secrets (GitHub App method)

release-please is given a **GitHub App token** so the setup can be reused across many repositories.
(The default `GITHUB_TOKEN` cannot trigger downstream workflows for actions it performs — so CI on the Release PR and `on: release`-driven follow-up steps would not fire. A personal PAT is tied to a person and expires, which scales poorly. A GitHub App needs no organization, works on personal repos, and its installation token auto-expires/rotates hourly.)

### Setup

1. GitHub → **Settings (personal) → Developer settings → GitHub Apps → New GitHub App**
   - Repository permissions: **Contents = Read and write** / **Pull requests = Read and write**
   - Private (not public) is fine if you only use it on your own repos
2. After creating it, note the **App ID** and generate a **Private key** (download the `.pem`)
3. **Install** the App on the target repositories (the personal repos you want to roll this out to)
4. In each repo, add under **Settings → Secrets and variables → Actions**:
   - `APP_ID`: the App ID
   - `APP_PRIVATE_KEY`: the full contents of the generated `.pem`
5. In **Settings → Actions → General**, enable **Read and write permissions** and **Allow GitHub Actions to create and approve pull requests**

The `actions/create-github-app-token` step in the workflow mints a short-lived token from these two secrets and passes it to release-please. To roll this out to more repos, install the same App on each repo and add the same secret values (one private key, reused everywhere).

## Note: enabling actual Maven Central publishing

This sample stops at attaching the jar to a GitHub Release; it does not publish to Maven Central. To publish for real:

- Register a namespace on the [Central Portal](https://central.sonatype.com/) (for GitHub-based identities, `io.github.<user>`) and generate a Portal token.
- Add `central-publishing-maven-plugin` (`extensions=true`) plus `maven-gpg-plugin` / `maven-source-plugin` / `maven-javadoc-plugin` to `pom.xml`.
- In the `publish` job, pass the GPG private key and Portal token from secrets and switch to `mvn deploy`.
