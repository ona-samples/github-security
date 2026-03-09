# GitHub Security + Ona Automations

Security scanners are good at finding vulnerabilities. Fixing them is the hard part. Teams accumulate a backlog of findings that grows faster than developers can address it. Developers see security fixes as toil. And tools that auto-generate PRs via text search-and-replace often produce changes that are insufficiently tested or break the application — creating more work, not less.

[Ona](https://ona.com) takes a different approach:

- An **AI software engineer** analyzes each finding and crafts the code change — not a regex substitution, but a reasoned fix that accounts for the project's structure and conventions.
- The fix is **built and tested in a fully equipped dev environment** where the modified code can actually compile and run.
- The agent **iterates until the fix is proven not to break the app** — if tests fail, it reads the errors, adjusts, and retries.

The result is a PR that is ready to review and merge, not a starting point that needs manual cleanup.

This repo demonstrates the setup using [Spring PetClinic](https://github.com/spring-projects/spring-petclinic) (Java/Maven) with GitHub's free security features.

## Security scanning tools

All tools below are free for public repos on GitHub's free org plan.

### Dependabot alerts

Dependabot monitors your dependency graph for known vulnerabilities and creates alerts under **Security > Dependabot**.

For Maven projects, GitHub's dependency graph often can't resolve versions inherited from a parent BOM. The [`dependency-submission.yml`](.github/workflows/dependency-submission.yml) workflow solves this by running `mvn` to resolve the full dependency tree and submitting it to GitHub's dependency graph API.

### Code scanning (CodeQL)

[CodeQL](https://codeql.github.com/) performs static analysis on your source code. GitHub's default setup analyzes Java and Actions code on every push and PR. Results appear under **Security > Code scanning**.

### Trivy (filesystem scan)

[Trivy](https://github.com/aquasecurity/trivy) scans dependency files (pom.xml, lock files, etc.) for known CVEs. The [`trivy.yml`](.github/workflows/trivy.yml) workflow runs a filesystem scan and uploads SARIF results to **Security > Code scanning**.

### OSV-Scanner

[OSV-Scanner](https://github.com/google/osv-scanner) checks dependencies against the [OSV database](https://osv.dev/). The [`osv-scanner.yml`](.github/workflows/osv-scanner.yml) workflow runs on push (scheduled scan) and on PRs (diff scan to catch newly introduced vulnerabilities). Results upload to **Security > Code scanning**.

## Ona automations

Two Ona automations in [`.ona/`](.ona/) use the GitHub CLI to fetch the highest-severity open alert, apply a fix, run tests, and open a PR.

### `fix-dependabot-alert`

[`.ona/fix-dependabot-alert.yaml`](.ona/fix-dependabot-alert.yaml)

1. **Install gh CLI** if not present
2. **Fetch** the highest-severity open Dependabot alert via `gh api`
3. **Analyze** the alert and read the manifest to understand how the dependency is declared
4. **Upgrade** the dependency to the patched version
5. **Verify** with `./mvnw compile test` and `./mvnw dependency:tree`
6. **Open a PR** with alert details, CVE, CVSS score, and verification checklist

### `fix-codescan-alert`

[`.ona/fix-codescan-alert.yaml`](.ona/fix-codescan-alert.yaml)

1. **Install gh CLI** if not present
2. **Fetch** the highest-severity open code scanning alert via `gh api`
3. **Analyze** the alert, read the affected source file and context
4. **Fix** the issue (code change for CodeQL findings, dependency upgrade for Trivy/OSV findings)
5. **Verify** with `./mvnw compile test`
6. **Open a PR** with alert details and verification checklist

Both automations authenticate using the token from the git credential helper (`GITHUB_TOKEN` env var), avoiding the need for additional secrets.

## Set up on your own repo

### 1. Enable security scanning

Set up scanners so that alerts appear under **Security** in your GitHub repo. This repo uses Dependabot, CodeQL, Trivy, and OSV-Scanner — see the [`.github/workflows/`](.github/workflows/) directory for examples. Use whichever combination fits your project.

GitHub docs:
- [Dependabot alerts](https://docs.github.com/en/code-security/dependabot/dependabot-alerts/configuring-dependabot-alerts)
- [Code scanning (CodeQL)](https://docs.github.com/en/code-security/code-scanning/enabling-code-scanning/configuring-default-setup-for-code-scanning)
- [Third-party SARIF uploads](https://docs.github.com/en/code-security/code-scanning/integrating-with-code-scanning/uploading-a-sarif-file-to-github) (Trivy, OSV-Scanner, etc.)

### 2. Add Ona automations

Copy the two automation files into your repo:

```
.ona/fix-dependabot-alert.yaml
.ona/fix-codescan-alert.yaml
```

Adjust the agent prompts if your project uses a different build tool (e.g., replace `./mvnw` with `./gradlew` or `npm`).

#### Prerequisites

Log in to Ona before running any `ona ai` commands:

```bash
ona login
```

#### Install automations

Use the Ona CLI to register each automation:

```bash
ona ai automation create .ona/fix-dependabot-alert.yaml
ona ai automation create .ona/fix-codescan-alert.yaml
```

#### Update automations

After editing a YAML file, update the registered automation. First find the automation ID:

```bash
ona ai automation list
```

Then apply the updated file:

```bash
ona ai automation update <automation-id> .ona/fix-dependabot-alert.yaml
```

#### Run automations

Trigger them manually from the Ona dashboard or via the CLI. Each run picks the highest-severity open alert, fixes it, and opens a PR.

## License

The Spring PetClinic sample application is released under version 2.0 of the [Apache License](https://www.apache.org/licenses/LICENSE-2.0).
