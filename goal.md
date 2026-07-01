# goal — `diene.dotnet-lib`

> A sample **NuGet library** template that proves the reusable AtomiCloud library
> patterns end-to-end, ready to be lifted into a CyanPrint template.
> Derived from [`diene.dotnet-base`](../dotnet-base); reference gold standard
> [`AtomiCloud/carboxylic.lithium`](https://github.com/AtomiCloud/carboxylic.lithium),
> upgraded to newest .NET + newest practice.

## 1. What this repo is

- A **publishable .NET library sample** — not a runnable service. It exists to
  prove that a library can go from source → packed → versioned → tagged →
  pushed to nuget.org, with all quality gates, using only additive changes on
  top of `dotnet-base`.
- The shipped code is **illustrative and swappable**. A downstream clones this,
  rips out the sample domain, drops in their library, and keeps the whole
  publish/CI/coverage machinery.
- It is the **.NET analogue of [`diene.bun-lib`](../bun-lib)** (base → publishable
  npm library). Same philosophy, dotnet mechanics.

## 2. Spirit to capture (from carboxylic.lithium)

- **One small idea, fully productionised.** carboxylic ships `AtomiCloud.Result`
  (a Result / Railway-Oriented-Programming type) — minimal domain, maximal
  lifecycle. Prove the *pipeline*, not a big library.
- **Multi-package from one repo.** Its signature shape is a **library + a
  companion test-helper package** (`AtomiCloud.Result` + `AtomiCloud.Result.TestHelper`,
  the latter referencing the lib and adding FluentAssertions steps). The sample
  must demonstrate shipping N coordinated packages in lockstep.
- **Conventional-commit-driven semver → publish.** Version is computed from
  commits, stamped into the package, tagged, and the tag triggers publish.
- **Reproducible spine.** Every CI step is `nix develop .#<shell> -c ./scripts/ci/<x>.sh`,
  runnable identically locally — inherited from base, keep it.
- **Package quality is a feature.** Branding (icon/logo), README-in-package,
  license, symbols, and high-bar coverage on the shipped surface.

## 3. High-level goals

### 3.1 Multi-library solution layout
- Keep base's 4-project shape; add a **second packable project** so the
  "lib + sibling test-helper" pattern is real, not hypothetical:
  - `Lib/` → the shipped library (packable).
  - `TestHelper/` → packable, `ProjectReference` → `Lib`, its own `PackageId`.
  - `UnitTest/`, `IntTest/` → unchanged in role.
  - `App/` → kept as a **"consumes-the-lib" demo**, marked `IsPackable=false`
    (keeping it is more additive/3wm-friendly than deleting it).
- Add the new project as a single additive line in `dotnet-base.slnx`.

### 3.2 Packaging metadata (shared where possible)
- Put **shared** packaging props once in `Directory.Build.props`
  (`Authors`, `Company`, `PackageProjectUrl`, `RepositoryUrl`, `RepositoryType`,
  `PackageLicenseExpression`, symbol/source-link/deterministic flags). Each
  packable csproj sets only `PackageId`, `Description`, `IsPackable`,
  `PackageReadmeFile`.
- **Version is not baked into the csproj.** With the tag-derived approach (§3.3),
  csproj carries only a dev placeholder (e.g. `VersionPrefix` `0.0.0`, or omit
  it); the real version is injected at pack time via `--property:Version=…`. Do
  not hand-maintain per-csproj versions.
- Bundle **docs + branding into the package**: `PackageReadmeFile` +
  `PackageIcon` (+ `LICENSE`, `logo.png` at root) via `<None Pack="true">`.

### 3.3 Release → NuGet publish (two-phase, tag-triggered)
- Reuse base's semver `release.yaml` (compute version from commits, cut a
  `v*.*.*` tag). **Add** a tag-triggered publish path:
  - `scripts/ci/publish.sh` — derive `VERSION="${GITHUB_REF_NAME#v}"`, then
    `dotnet pack <target> --property:Version=$VERSION` each comma-delimited
    target → `dotnet nuget push --skip-duplicate` (lib + `.snupkg` symbols),
    `NUGET_API_KEY` from secret, `nix develop .#cd -c`.
  - Version stamping across all packages in lockstep — all packed with the same
    tag-derived `$VERSION`, so no per-package version lives in git. **Call out the
    design choice**: carboxylic *commits* the csproj `<VersionPrefix>` bump
    (`update_version.sh` + `@semantic-release/git` assets); bun-lib *derives*
    version from `GITHUB_REF_NAME` and **never commits it**. Prefer the
    tag-derived, no-commit approach — cleaner history, smaller 3wm surface, and
    it removes the need for carboxylic's `update_version.sh` / xmlstarlet step.
  - `.github/workflows/cd.yaml` — add a `publish` job on `v*.*.*` (or a
    `reusable-publish.yaml`, bun-lib style).

### 3.4 CI: prove the artifact, not just the code
- Add a **`package-validate`** job (the dotnet analogue of bun-lib's
  publint/attw gate): `dotnet pack` + validate the nupkg (pack succeeds,
  symbols present, metadata complete, no missing README/icon). "Does it
  actually install as a package?" gate.
- Keep base's precommit / dead-code / unit / int / build jobs untouched.

### 3.5 Testing & coverage on the shipped surface
- Per-package targeted coverage via the existing `.config/*.test.yaml`
  mechanism — high bar (e.g. 100% unit on `[Lib]`) on shipped code.
- Sample tests exercise both `Lib` and `TestHelper`; `UnitTest` references
  `TestHelper` and uses its assertion helpers (carboxylic's precedent —
  the test-helper package is dogfooded by the repo's own tests).

### 3.6 Docs
- `README.md`: NuGet install snippet, usage, version/downloads badges
  (bun-lib README shape).
- `docs/developer/dotnet-lib-baseline.md` (parallel to bun's `bun-baseline.md`):
  pack/publish flow, per-package coverage, "this is a library not a service,"
  and the **promotion knobs** a downstream swaps (see §5).

## 4. Newest-practice upgrades (carboxylic is the "before")

- **Target `net10.0`** (base already does) — not net8.0.
- **Central Package Management** (base already has it) — fixes carboxylic's
  inline per-csproj version drift.
- **Source Link + deterministic + CI build**: `Microsoft.SourceLink.GitHub`,
  `PublishRepositoryUrl`, `EmbedUntrackedSources`, `ContinuousIntegrationBuild`
  (CI-gated), `Deterministic` (base has it) — debug straight into the package.
- **Symbol packages**: `IncludeSymbols` + `SymbolPackageFormat=snupkg`.
- **NuGet audit**: `NuGetAudit` + `NuGetAuditMode=all` for transitive CVE scan.
- **Real `RepositoryUrl`** (carboxylic literally shipped the string `"RepositoryUrl"`),
  promotion-rewritten like bun-lib rewrites its repo paths.
- **SPDX `PackageLicenseExpression`** (e.g. `MIT`) over `PackageLicenseFile`.
- **Keep base's modern test pins**: xunit.v3, Test.Sdk 18, FluentAssertions
  **7.x** (Apache-2.0 — v8 went commercial; carboxylic used v8), Testcontainers.

## 5. Promotion knobs (what a downstream template swaps)

- `PackageId` / `Description` per packable project.
- `Directory.Build.props`: `Authors`, `Company`, `RepositoryUrl`, project URL.
- Coverage thresholds in `.config/*.test.yaml`.
- README badges + install snippet (rewritten on promotion).
- The sample `Lib` / `TestHelper` source + tests (illustrative — replace with
  the real library).

## 6. Non-negotiable constraint — minimize the 3wm surface

- `main` here is an **exact copy of `dotnet-base`** (the bootstrap commit is the
  3-way-merge base). When upstream `dotnet-base` changes, downstream repos must
  merge cleanly.
- Therefore every change must be **additive and line-level**: new
  `PropertyGroup`/`ItemGroup` blocks, new files, single-line `.slnx`/props
  additions. Avoid rewrites, reordering, or reflowing base files.
- New behaviour lives in **new files** (`scripts/ci/publish.sh`, new workflows,
  `TestHelper/`) wherever possible, rather than edits to shared base files.

## 7. Out of scope

- Runtime service concerns (hosting, HTTP, deployment) — that's `dotnet-api`.
- Actual implementation of the library logic beyond a minimal illustrative
  sample — the point is the pattern and the pipeline.
