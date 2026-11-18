# Upgrade summary: net6.0–net9.0 → net10.0

## Result
- `dotnet build` succeeds with 0 errors and 0 warnings for all seven upgraded projects (Giraffe library, five samples, Giraffe.Tests). The Giraffe library builds with `TreatWarningsAsErrors=true`.
- `dotnet test`: 454 tests run, 454 pass, none skipped.

## Changes
| File | Change |
|---|---|
| `src/Giraffe/Giraffe.fsproj` | `TargetFrameworks` `net6.0;net7.0;net8.0;net9.0` → `TargetFramework` `net10.0`. Removed the `System.Text.Json` 8.0.* reference because the net10.0 shared framework supplies it. `Microsoft.SourceLink.GitHub` 8.0.* → 10.0.* because 8.0 pulls in `Microsoft.Build.Tasks.Git` 8.0.0, which has advisory GHSA-23fw-v26w-5fgq (NU1902, which fails the build as a warning-as-error). Kept the `FSharp.Core` 6.0.0 pin and `DisableImplicitFSharpCoreReference`, and kept the compile order unchanged. |
| `samples/*/*.fsproj` (5 projects) | `net8.0`/`net9.0` → `net10.0`. |
| `tests/Giraffe.Tests/Giraffe.Tests.fsproj` | `net8.0;net9.0;net10.0` → `net10.0`. The three per-TFM `Microsoft.AspNetCore.TestHost` groups are now one unconditional 10.0.* reference. |
| `src/Giraffe/Core.fs`, `src/Giraffe/HttpContextExtensions.fs` | Removed the dead `#else` branch of `#if NET8_0_OR_GREATER` in `markdown` / `WriteMarkdownAsync`. They now always use `MediaTypeNames.Text.Markdown`, which is the same `text/markdown` value. |
| `tests/Giraffe.Tests/Helpers.fs` | Removed the dead pre-net10 `WebHostBuilder`/`TestServer` branches. The `HostBuilder` + `UseTestServer` path that was already used on net10.0 is kept. |
| `samples/GlobalRateLimiting/Program.fs` | `TimeSpan.FromSeconds(1)` → `TimeSpan.FromSeconds(int64 1)`. This fixes FS0041: .NET 9+ added `int64` overloads, so an `int` literal is ambiguous. It follows the pattern the RateLimiting sample already uses. |

No System.Web, EF6 or other .NET Framework APIs were present, so those KB steps did not apply.

## Next steps
- **netstandard2.0 not added to Giraffe:** the library is not multi-targeted to netstandard2.0. It depends on `<FrameworkReference Include="Microsoft.AspNetCore.App" />` and ASP.NET Core APIs throughout (HttpContext, endpoint routing, rate limiting), and none of these exist on netstandard2.0. The package also never targeted netstandard2.0 before, so there are no .NET Framework consumers to keep.
- **Package consumers:** dropping net6.0–net9.0 from the Giraffe NuGet package is a breaking change. Consumers on .NET 8/9 can no longer take new versions. If the package has to keep supporting them, add `net8.0;net9.0` back next to `net10.0` and restore the matching `#if` branches. Record the decision in CHANGELOG.md / RELEASE_NOTES.md.
- **CI:** `.github/workflows/*.yml` still install the 8.x and 9.x SDKs. Only 10.x is needed now, so the others can be dropped (optional).
- **FSharp.Core:** still pinned at 6.0.0 for the library, as the KB guidance for published libraries recommends. Raise it only if newer FSharp.Core APIs are needed.
