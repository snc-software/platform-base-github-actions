# platform-base-github-actions

Shared, reusable GitHub Actions workflows and composite actions for .NET repositories in this
organization. Individual repos call these instead of maintaining their own CI/CD pipelines, so a
fix or improvement made here rolls out everywhere that references it.

## Pinning: the `actionsTag` input

Every reusable workflow takes an `actionsTag` input (default `"main"`). It controls which
branch/tag of **this** repo is checked out to provide the composite actions (GitVersion, format
check, NuGet pack/push, etc.) used internally by the workflow run.

- Leaving it at `main` means the caller always gets the latest version of these composite actions —
  intentional, since `main` here is meant to flow to every downstream repo without requiring each
  one to bump a pin.
- Set it to a feature branch (e.g. `actionsTag: my-experiment`) to try a pre-release change to this
  actions repo from a single downstream repo before merging it to `main`.

Note this only affects the composite actions resolved via the `.platform-actions` checkout inside
the job. The `@main` in `uses: snc-software/platform-base-github-actions/.github/workflows/...@main`
in the caller's own workflow file is a separate pin — the caller decides which version of the
*workflow itself* to call, this ref is the one that decides which version of the workflow steps
run at all.

## Workflows

### `build-dotnet-library.yml`

Build, test, and publish a .NET library as a NuGet package.

```yaml
jobs:
  build:
    uses: snc-software/platform-base-github-actions/.github/workflows/build-dotnet-library.yml@main
    with:
      buildConfiguration: Release   # optional, default "Release"
      codeCoverageThreshold: 80     # optional, default 80
      actionsTag: main              # optional, default "main"
    secrets: inherit
```

| Input | Type | Default | Description |
|---|---|---|---|
| `buildConfiguration` | string | `Release` | Build configuration passed to `dotnet build`/`test`. |
| `codeCoverageThreshold` | number | `80` | Minimum line coverage percentage; the job fails below this. |
| `actionsTag` | string | `main` | Branch/tag of this repo to source composite actions from — see above. |

Pipeline:
1. Checks out the caller repo (full history) and this actions repo.
2. Runs GitVersion (Mainline mode) to compute the version for this build.
3. Restores, builds, and runs `dotnet test` with Coverlet coverage collection.
4. Publishes test results and a coverage report, and fails the job if coverage is below
   `codeCoverageThreshold`.
5. Packs and pushes the library to GitHub Packages as a NuGet package — pre-release on PR builds,
   release on `main`.
6. On `main` only, tags the commit with the computed version and creates a GitHub Release.

The computed `semver`/`nuget-version` are exposed as job outputs on the internal build job for use
by the release step in this workflow, but are **not** currently surfaced to the caller via
`workflow_call.outputs` — a caller can't read the version this workflow computed today.

### `build-dotnet-service.yml`

Build, test, and optionally publish a .NET API service (and its NuGet-packaged clients, if any).

```yaml
jobs:
  build:
    uses: snc-software/platform-base-github-actions/.github/workflows/build-dotnet-service.yml@main
    with:
      buildConfiguration: Release        # optional, default "Release"
      codeCoverageThreshold: 80          # optional, default 80
      buildPackages: true                # optional, default false — pack/push NuGet packages (e.g. API clients)
      bundleSQLMigrations: true          # optional, default false — bundle a raw SQL migrations directory as an artifact
      migrationsDirectory: "./migrations" # required if bundleSQLMigrations is true
      actionsTag: main                   # optional, default "main"
    secrets: inherit
```

| Input | Type | Default | Description |
|---|---|---|---|
| `buildConfiguration` | string | `Release` | Build configuration passed to `dotnet build`/`test`. |
| `codeCoverageThreshold` | number | `80` | Minimum line coverage percentage; the job fails below this. |
| `buildPackages` | boolean | `false` | Whether to pack and push NuGet packages from the solution (e.g. `Api.Clients`, `Events`). |
| `bundleSQLMigrations` | boolean | `false` | Whether to bundle a raw SQL migrations directory as a build artifact. |
| `migrationsDirectory` | string | `""` | Path to the raw SQL migrations directory to copy. Required (and must not resolve to `/`) when `bundleSQLMigrations` is `true`. |
| `actionsTag` | string | `main` | Branch/tag of this repo to source composite actions from — see above. |

Same build/test/coverage/versioning/release pipeline as the library workflow, plus:
- Optional SQL migration bundling as a downloadable artifact (raw file copy only — this does not
  run `dotnet ef migrations bundle` or any EF Core-specific packaging).
- NuGet packing/pushing is opt-in via `buildPackages` rather than always-on, since most services
  don't publish a package themselves.

## Composite actions

Used internally by the two workflows above; not typically referenced directly by consumers.

- `actions/gitversion` — runs [GitVersion](https://github.com/GitTools/actions) in Mainline mode
  and exposes `FullSemVer`/`NuGetVersion`/`SemVer` outputs.
- `actions/dotnet-format-check` — runs `dotnet format --verify-no-changes`.
- `actions/build-push-nuget` — packs and pushes NuGet packages to the GitHub Packages feed.
- `actions/bundle-sql-migrations` — copies a raw SQL migrations directory and uploads it as an
  artifact.
