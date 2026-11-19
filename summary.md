# .NET → net10.0 transformation — Giraffe (F#)

## Target

All seven upgraded projects now target **`net10.0`** only, replacing the previous
`net6.0;net7.0;net8.0;net9.0` / `net8.0;net9.0;net10.0` / `net8.0` / `net9.0` mix.

## Result

| Check | Outcome |
| --- | --- |
| `dotnet restore Giraffe.slnx` | success, 0 errors, 0 warnings |
| `dotnet build Giraffe.slnx` (Debug) | **success, 0 errors, 0 warnings** |
| `dotnet build Giraffe.slnx -c Release` | **success, 0 errors, 0 warnings** |
| `dotnet build` on each of the 7 projects individually | exit code 0, 0 errors, 0 warnings each |
| `dotnet test -c Release` | **454 passed, 0 failed, 0 skipped** |
| `dotnet pack src/Giraffe -c Release` | `Giraffe.8.4.0.nupkg` + symbols produced |
| `dotnet fantomas --check src samples tests` | exit code 0 (formatting clean) |
| `dotnet msbuild /t:AnalyzeFSharpProject src/Giraffe/Giraffe.fsproj` | runs, no assembly-load failures |

No test was deleted, skipped, disabled or weakened. The test count and assertions are unchanged
from the pre-upgrade project.

## Changes

### Project files (target framework)

- `src/Giraffe/Giraffe.fsproj` — `<TargetFrameworks>net6.0;net7.0;net8.0;net9.0</TargetFrameworks>`
  → `<TargetFramework>net10.0</TargetFramework>`.
- `tests/Giraffe.Tests/Giraffe.Tests.fsproj` — `<TargetFrameworks>net8.0;net9.0;net10.0</TargetFrameworks>`
  → `<TargetFramework>net10.0</TargetFramework>`.
- `samples/NewtonsoftJson`, `samples/RateLimiting`, `samples/EndpointRoutingApp`,
  `samples/ResponseCachingApp` — `net9.0` → `net10.0`;
  `samples/GlobalRateLimiting` — `net8.0` → `net10.0`.

Every `.fsproj` keeps its `<Compile Include>` items in the original order, and the F#-specific
settings were left as they were (`EnableDefaultCompileItems=false`,
`DisableImplicitFSharpCoreReference=true`, `TreatWarningsAsErrors=true`,
`NoWarn=FS2003;FS0044`).

### Packages

- `tests/Giraffe.Tests` — the three per-TFM `<ItemGroup Condition="'$(TargetFramework)' == '...'">`
  blocks for `Microsoft.AspNetCore.TestHost` (8.0.\*, 9.0.\*, 10.0.\*) collapsed into a single
  unconditional `10.0.*` reference, matching the single remaining target.
- `src/Giraffe` — removed `<PackageReference Include="System.Text.Json" />`. On `net10.0` this
  package ships in the shared framework, and NuGet raised **NU1510** ("will not be pruned …
  remove the PackageReference item"), which `TreatWarningsAsErrors` turned into a restore
  **error**. `FSharp.SystemTextJson` still brings in the API surface it needs via the framework.
- `src/Giraffe` — removed `<PackageReference Include="Microsoft.SourceLink.GitHub" Version="8.0.*" />`.
  It transitively pulled `Microsoft.Build.Tasks.Git` 8.0.0, which NuGet audit flagged with
  **NU1902** (GHSA-23fw-v26w-5fgq, moderate) — again an error under `TreatWarningsAsErrors`.
  SourceLink has been part of the .NET SDK since .NET 8, so source-link metadata is still
  generated; `PublishRepositoryUrl` and `EmbedUntrackedSources` are untouched and
  `dotnet pack` still produces the symbols package.
- `FSharp.Core` stays pinned at the explicit `6.0.0` with `DisableImplicitFSharpCoreReference`.
  Per the F# guidance for a library published as a NuGet package, the pin is deliberately low so
  it does not become a floor for every consumer, and nothing in the net10.0 build needs a newer
  FSharp.Core.

### Code

- `samples/GlobalRateLimiting/Program.fs:33` — `TimeSpan.FromSeconds(1)` → `TimeSpan.FromSeconds(float 1)`.
  .NET 9 added `TimeSpan.FromSeconds(int64)` and `FromSeconds(int64, ?int64, ?int64)` overloads, so
  the untyped integer literal became ambiguous and failed with **FS0041**. The `float` annotation
  pins it back to the overload it resolved to before, so the window is still exactly one second.
  This matches the style already used elsewhere in the repo (`float 30`, `int64 12`).
- `src/Giraffe/Core.fs` and `src/Giraffe/HttpContextExtensions.fs` — dropped the now-dead
  `#if NET8_0_OR_GREATER` / `#else` branches around the markdown content type, keeping the
  `NET8_0_OR_GREATER` side (`System.Net.Mime.MediaTypeNames.Text.Markdown + "; charset=utf-8"`).
  With `net6.0`/`net7.0` gone, the `#else` branch could never compile. The emitted content type
  is byte-identical (`text/markdown; charset=utf-8`).
- `tests/Giraffe.Tests/Helpers.fs` — dropped the now-dead `#if NET10_0_OR_GREATER` / `#else`
  branches in `createHost` and `makeRequest`, keeping the `net10.0` side (`HostBuilder` +
  `ConfigureWebHost` + `UseTestServer`, then `host.StartAsync()` / `host.GetTestServer()`).
  The pre-net10 `WebHostBuilder` / `new TestServer(...)` path is unreachable now that `net10.0`
  is the only target.

### Build and CI tooling

- `.github/workflows/build-and-test.yml` and `.github/workflows/publish.yml` — the
  `actions/setup-dotnet` SDK list `8.x / 9.x / 10.x` reduced to `10.x`, since no project
  targets .NET 8 or .NET 9 any more.
- `.config/dotnet-tools.json` + `src/Giraffe/Giraffe.fsproj` — the F# analyzer stack was
  mutually incompatible (a pre-existing problem, independent of the retarget): the
  `fsharp-analyzers` CLI 0.38.0 refused to load `Ionide.Analyzers` 0.20.0
  ("built using SDK version 0.41.0.0. Expect 0.38.0.0 instead. Assembly will be skipped"),
  and then bailed out with "Because we failed to load some assemblies … exiting". Because
  `FSharp.Analyzers.SDK` requires an exact version match between the CLI and every analyzer
  pack, the three were aligned on SDK **0.41.0**: CLI `0.38.0 → 0.41.0`,
  `G-Research.FSharp.Analyzers` `0.24.0 → 0.26.0`, `Ionide.Analyzers` kept at `0.20.0`.
  `AnalyzeFSharpProject` now loads both packs and completes without load errors.
  (The newest packs cannot be used together: `G-Research.FSharp.Analyzers` 0.27.0 is built
  against SDK 0.41.1 while `Ionide.Analyzers` 0.21.0/0.22.0 need 0.42.0/0.43.0.)

## Next steps

Nothing blocks the build or the tests. The items below are advisory and left for human review.

1. **Analyzer advisories from `AnalyzeFSharpProject` (not from `dotnet build`).** The
   warning-driving analyzer step — which CI runs separately with `continue-on-error: true` —
   still reports four pre-existing `GRA-*` warnings plus a set of `IONIDE-*` info/hint
   diagnostics. `dotnet build` itself is at **0 warnings**, so none of these gate the build.
   They were deliberately not "fixed" because the fixes are behavioural or stylistic product
   decisions rather than upgrade work:
   - `GRA-STRING-002` at `src/Giraffe/ModelParser.fs:232`, `src/Giraffe/Core.fs:262`,
     `src/Giraffe/Routing.fs:242` — `String.StartsWith(string)` with no `StringComparison`.
     Adding an explicit overload is the recommended fix, but `StartsWith(string)` uses
     *current-culture* comparison while `StringComparison.Ordinal` does not. For route and
     form-key matching ordinal is almost certainly the intended semantics, yet changing it
     silently alters matching behaviour in a widely consumed web framework. Needs a maintainer
     decision (and ideally a test) before being changed.
   - `GRA-TYPE-ANNOTATE-001` at `src/Giraffe/Routing.fs:12` — annotate the type passed to the
     `string` function.
   - `IONIDE-002` (postfix array/seq/list syntax), `IONIDE-004` (unnamed union-case fields),
     `IONIDE-010` (`Seq.filter |> Seq.map` → `Seq.choose`), `IONIDE-012` (`[<Struct>]` on
     discriminated unions). `IONIDE-004` and `IONIDE-012` touch public union types, which
     changes the surface C# consumers see, so they are not safe drive-by edits in a retarget.
2. **Analyzer packs are pinned one minor behind the latest.** `G-Research.FSharp.Analyzers`
   0.26.0 / `Ionide.Analyzers` 0.20.0 / CLI 0.41.0 is the newest *mutually loadable*
   combination today. Once `G-Research.FSharp.Analyzers` ships a build against
   `FSharp.Analyzers.SDK` 0.42/0.43, all three can move to the latest together.
3. **`dotnet tool restore` reports a newer `fantomas` (8.1.2 vs the pinned 8.1.1).** Left pinned
   — a formatter bump can reformat the whole tree and would bury the upgrade diff. Worth doing
   as its own commit.
4. **`FSharp.Core` is still pinned at 6.0.0** for the published library, by design (see above).
   If the maintainers would rather track the SDK's FSharp.Core, remove both the pin and
   `DisableImplicitFSharpCoreReference` together — but note that raising it raises the floor for
   every downstream consumer of the Giraffe package.
5. **`samples/RateLimiting` uses `Microsoft.NET.Sdk`, not `Microsoft.NET.Sdk.Web`**, unlike the
   other four samples, and relies on the `Microsoft.AspNetCore.App` framework reference coming
   in through the `Giraffe` project reference. That is pre-existing and it builds, so it was left
   alone; switching it to the Web SDK would be a tidy-up for a separate change.
6. **Multi-targeting was removed, not narrowed.** `Giraffe` is a published NuGet package, and it
   now ships a single `net10.0` asset instead of `net6.0`–`net9.0`. That is what this
   transformation asked for, but it is a consumer-visible packaging change: anyone still on
   .NET 6–9 can no longer take a new Giraffe release. The `CHANGELOG.md` / `RELEASE_NOTES.md`
   entry and the package's major version bump are a release decision and were not written here.
