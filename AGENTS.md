# Guidance for AI agents integrating Mauro CI

These instructions are for agents changing a *consuming repository* to use
the shared workflows or composite actions in `MauroDataMapper/mauro-ci`.
Copy this file to the consuming repository's root `AGENTS.md`, or provide it
to the agent as task context. The repository's `README.md` is the user-facing
overview; the workflow and action files are the source of truth for inputs,
secrets, permissions, and behavior.

## Before making changes

- Inspect the consuming repository's existing workflows, triggers, default
  branch, project scripts or tasks, artifact names, required checks, and
  permissions.
- Reuse a shared workflow or action rather than copying its implementation.
- Use a released version or immutable commit SHA in each `uses` reference.
  Replace `<ref>` in examples; do not use `@main` for production workflows.
- Confirm each input and secret against the selected workflow's
  `workflow_call` declaration. Do not invent inputs or pass unrelated secrets.
- Preserve existing CI behavior unless the task explicitly asks to change it.

## Choosing a workflow or action

| Need | Reuse | Important behavior |
| --- | --- | --- |
| Run Gradle tests | `.github/workflows/gradle-test.yml` | Checks out the consuming repository, defaults to Java 17 and the `test` task, and uploads `junit-results`. |
| Publish Gradle artifacts | `.github/workflows/gradle-publish.yml` | `target: snapshot` only publishes on `develop`; `target: maven-central` only publishes on `main`. Pass signing and repository secrets. |
| Run Node.js CI | `.github/workflows/node-ci.yml` | Runs `npm ci` and named npm scripts with no secrets and `contents: read`; report and build artifact paths are relative to the repository root. |
| Set up Node.js in a custom job | `actions/setup-node-npm` | Configures Node.js/npm and installs the lockfile dependencies; use when a repository needs project-specific setup steps in the same job. |
| Link a local npm dependency | `actions/link-npm-package` | Checks out and runs install/build scripts for a public dependency; use only in jobs without secrets. |
| Publish Node.js artifacts | `.github/workflows/node-publish.yml` | Downloads an artifact without checking out or running repository code; publishes snapshots only on `main`/`develop` pushes and optional releases on `main`, using a required caller-configured environment for protection rules. |
| Validate a Docker build | `.github/workflows/docker-build.yml` | Builds a multi-architecture image without publishing it. |
| Publish to Docker Hub | `.github/workflows/docker-publish.yml` | Publishes only on pushes to `main`; pass Docker Hub credentials. |
| Lint with MegaLinter | `.github/workflows/megalinter.yml` | The checked-in workflow/configuration supports Groovy. `scan-mode` is `changed` by default or can be `full`; it uploads `megalinter-reports`. Set `scan-pr-commits: true` to enable Gitleaks scanning between PR base and head commits. |
| Publish JUnit results | `.github/workflows/junit-report.yml` | Expects the `junit-results` artifact and only runs for same-repository pull requests or pushes to `main`/`develop`. |
| Upload SARIF | `.github/workflows/sarif-upload.yml` | Expects `megalinter-reports` by default, accepts another artifact and relative file path, and only uploads on pushes to `main`/`develop`. |
| Send a Zulip summary | `.github/workflows/zulip-notify.yml` | Pass job results and `ZULIP_API_KEY`; use `if: always()` if it must run after failed or cancelled jobs. |
| Configure Java and Gradle | `actions/setup-gradle-java` | Composite action; use it as a step and provide `java-version` if the default is unsuitable. |
| Configure Node.js/npm | `actions/setup-node-npm` | Composite action; requires a lockfile and runs `npm ci`. |
| Link a public npm dependency | `actions/link-npm-package` | Composite action; runs package installation/build scripts and links the package into the current project. |
| Configure Docker Buildx | `actions/setup-docker-buildx` | Composite action; use it as a step before a multi-architecture build. |
| Install shared MegaLinter config | `actions/prepare-megalinter` | Composite action; currently supports Groovy `changed` and `full` configurations. |

Publishing workflows contain their own branch/event conditions. A successful
call outside those conditions may be skipped rather than publish anything.
Do not weaken those guards to make an example run; confirm the intended
release branch and GitHub environment first.

## Calling reusable workflows

Call reusable workflows at the job level with the fully qualified repository
path. For example:

```yaml
jobs:
  test:
    uses: MauroDataMapper/mauro-ci/.github/workflows/gradle-test.yml@<ref>
    with:
      java-version: "17"
      gradle-task: test
```

Composite actions are used in a job's `steps`:

```yaml
steps:
  - uses: MauroDataMapper/mauro-ci/actions/setup-gradle-java@<ref>
    with:
      java-version: "17"
```

When reporting results, make the notification depend on the jobs whose
statuses it reports and ensure it is not skipped after failures:

```yaml
  notify:
    if: always()
    needs: [test, lint]
    uses: MauroDataMapper/mauro-ci/.github/workflows/zulip-notify.yml@<ref>
    with:
      workflow-name: CI
      build-result: ${{ needs.test.result }}
      lint-result: ${{ needs.lint.result }}
    secrets:
      ZULIP_API_KEY: ${{ secrets.ZULIP_API_KEY }}
```

## Permissions, secrets, and artifacts

- A called workflow cannot increase the `GITHUB_TOKEN` permissions granted by
  its caller. The JUnit report workflow needs `actions: read`, `checks: write`,
  and `contents: read`. SARIF upload needs `actions: read`, `contents: read`,
  and `security-events: write`. Grant only what the selected workflow needs.
- Pass only required secrets by name. Secrets are not available to workflows
  triggered by fork pull requests; guard secret-dependent calls accordingly.
- `node-ci.yml` runs repository npm scripts, so it must not receive secrets or
  write-capable tokens. `node-publish.yml` is limited to trusted push events,
  does not check out or execute repository code, and binds publishing to the
  caller-configured environment. Configure protection rules and credentials in
  that environment, or explicitly map repository credentials.
- JUnit and SARIF reporting consume artifacts produced earlier in the same
  workflow run. Preserve the expected artifact names or pass the corresponding
  input when a workflow supports one.
- Set up required repository or environment secrets before enabling publishing
  or Zulip notifications. Never put credentials in workflow inputs or files.

## Validation

After editing a consumer workflow, run `actionlint` and the repository's
existing workflow/security checks. If the consumer repository maintains an
Actions lockfile, update and verify it using that repository's documented
process.
