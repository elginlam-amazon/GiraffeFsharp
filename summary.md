# .NET 10 upgrade summary

All seven upgraded projects now target `net10.0` only.

## Changes

| File | Change |
| --- | --- |
| `src/Giraffe/Giraffe.fsproj` | `TargetFrameworks` `net6.0;net7.0;net8.0;net9.0` → `TargetFramework` `net10.0`. Removed `System.Text.Json` 8.0.* (the net10.0 shared framework provides it). Removed `Microsoft.SourceLink.GitHub` 8.0.*: it pulled in `Microsoft.Build.Tasks.Git` 8.0.0, which has a known vulnerability (NU1902, GHSA-23fw-v26w-5fgq) that `TreatWarningsAsErrors` turned into an error. The .NET 8+ SDK includes Source Link for GitHub, and `PublishRepositoryUrl` / `EmbedUntrackedSources` still apply. `FSharp.Core` 6.0.0 pin and `DisableImplicitFSharpCoreReference` kept. `<Compile Include>` order unchanged. |
| `src/Giraffe/Core.fs`, `src/Giraffe/HttpContextExtensions.fs` | Removed the `#if NET8_0_OR_GREATER` / `#else` branches, which no target uses any more. The code always uses `MediaTypeNames.Text.Markdown`, which has the same value `text/markdown`. |
| `tests/Giraffe.Tests/Giraffe.Tests.fsproj` | `net8.0;net9.0;net10.0` → `net10.0`. The per-TFM `Microsoft.AspNetCore.TestHost` groups are merged into one unconditional 10.0.* reference. |
| `tests/Giraffe.Tests/Helpers.fs` | Removed the `#if NET10_0_OR_GREATER` / `#else` branches and kept the net10.0 `HostBuilder` + `UseTestServer` path. No test assertions changed. |
| `samples/*/…fsproj` (NewtonsoftJson, RateLimiting, EndpointRoutingApp, GlobalRateLimiting, ResponseCachingApp) | `net8.0` / `net9.0` → `net10.0`. |
| `samples/GlobalRateLimiting/Program.fs` | `TimeSpan.FromSeconds(1)` → `TimeSpan.FromSeconds(1.0)`. .NET 9+ added `FromSeconds(int64)` overloads, so the int literal was ambiguous (FS0041). The window is still 1 second. |

## Verification

- `dotnet build` on each of the 7 projects: succeeded.
- `dotnet build Giraffe.slnx` (Debug) and `dotnet build -c Release Giraffe.slnx`: 0 errors, 0 warnings.
- `dotnet test --project tests/Giraffe.Tests/Giraffe.Tests.fsproj`: 454 passed, 0 failed, 0 skipped.

## Next steps

- **No netstandard2.0 leg on `Giraffe`.** The library was not multi-targeted to netstandard2.0 before this cycle, and it can't be: it depends on `<FrameworkReference Include="Microsoft.AspNetCore.App" />`, which is only available on `netcoreapp3.0+`. It ships as a single `net10.0` target.
- **Breaking change for package consumers.** Dropping net6.0–net9.0 from the published `Giraffe` NuGet package means consumers on .NET 8/9 can no longer use new versions. Confirm this is intended before the next release, and add an entry to `CHANGELOG.md` / `RELEASE_NOTES.md` (not edited in this cycle).
- **CI.** `.github/workflows/build-and-test.yml` still installs SDKs 8.x, 9.x and 10.x. Only 10.x is needed now. The extra SDKs do no harm and were left alone.
- **Docs.** README.md / DOCUMENTATION.md may still list older supported frameworks. Review them.
- **Analyzers.** The FSharp.Analyzers tool (`dotnet tool restore` + the `analyzers` CI job) was not run in this cycle.
