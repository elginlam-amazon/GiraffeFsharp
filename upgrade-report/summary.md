# Executive Report: .NET Version Upgrade — Giraffe F# Web Framework  
  
**Project:** giraffe-fsharp/Giraffe (functional ASP.NET Core web framework for F#)  
**Date:** 2026-11-18  
**Upgrade:** net6.0 / net7.0 / net8.0 / net9.0 → **net10.0**  
**Outcome:** ✅ Complete success — 0 errors, 0 warnings, 454/454 tests passing  
  
---  
  
## 1. 📋 Executive Summary  
  
The Giraffe F# web framework and all associated sample applications and test projects were successfully upgraded from their previous multi-target and mixed-framework configurations to a unified **net10.0** target. The upgrade touched 7 F# projects across the library core, 5 sample applications, and 1 test suite. The .NET Upgrade Assistant attempted to process all projects but was unable to do so due to a known limitation — it only supports C# and VB.NET projects, not F#. Claude CLI took over and completed the full migration autonomously in a single cycle, resolving three distinct compatibility issues introduced by .NET 10's new APIs and stricter package auditing, with every test passing on first run after the fix.  
  
---  
  
## 2. 🏗️ Application Changes  
  
- 🔄 **Giraffe core library** (`src/Giraffe/Giraffe.fsproj`): dropped 4 legacy target frameworks (`net6.0`, `net7.0`, `net8.0`, `net9.0`) in favour of a single `net10.0` target  
- 🧪 **Test project** (`tests/Giraffe.Tests/Giraffe.Tests.fsproj`): narrowed from `net8.0;net9.0;net10.0` to `net10.0` only  
- 🖥️ **5 sample applications** (EndpointRoutingApp, RateLimiting, GlobalRateLimiting, NewtonsoftJson, ResponseCachingApp): each bumped from either `net8.0` or `net9.0` to `net10.0`  
- 📦 **NuGet package updates**: removed a now-redundant `System.Text.Json` explicit reference (absorbed into the shared ASP.NET Core framework on net10.0); upgraded `Microsoft.SourceLink.GitHub` from `8.0.*` to `9.0.*`  
- 🩹 **Source code fix** in `GlobalRateLimiting/Program.fs`: resolved a .NET 10 overload-resolution ambiguity in `TimeSpan.FromSeconds`  
  
---  
  
## 3. 🛠️ Tools Used  
  
- 🔧 **.NET SDK 10.0.404** — build toolchain and F# 10 compiler used for all compilation and test execution  
- 🤖 **Microsoft .NET Upgrade Assistant v1.0.518.26217** — attempted automated project file upgrades; could not process F# projects (limitation: C# and VB.NET only); project ordering and scaffolding still provided useful input  
- 🧠 **Claude CLI v2.1.266 (Claude Code)** — model `global.anthropic.claude-sonnet-4-6` — performed all actual migration edits, diagnosed build errors, and validated the complete upgrade; ran 40 reasoning turns with tool use; total cost $0.83 USD  
  
---  
  
## 4. 💻 Code Changes  
  
### Project files (`.fsproj`)  
  
| File | Change |  
|---|---|  
| `src/Giraffe/Giraffe.fsproj` | `TargetFrameworks`: `net6.0;net7.0;net8.0;net9.0` → `net10.0` |  
| `src/Giraffe/Giraffe.fsproj` | Removed `PackageReference` for `System.Text.Json` (NU1510 — now platform-provided) |  
| `src/Giraffe/Giraffe.fsproj` | `Microsoft.SourceLink.GitHub`: `8.0.*` → `9.0.*` |  
| `src/Giraffe/Giraffe.fsproj` | Added `NU1902` to `<NoWarn>` (GHSA-23fw-v26w-5fgq affects all current `Microsoft.Build.Tasks.Git` versions; SourceLink is build-time only via `PrivateAssets="All"`) |  
| `tests/Giraffe.Tests/Giraffe.Tests.fsproj` | `TargetFrameworks`: `net8.0;net9.0;net10.0` → `net10.0` |  
| `tests/Giraffe.Tests/Giraffe.Tests.fsproj` | Collapsed 3 conditional `Microsoft.AspNetCore.TestHost` ItemGroups into one unconditional `10.0.*` reference |  
| All 5 sample `.fsproj` files | `TargetFramework`: `net8.0` / `net9.0` → `net10.0` |  
  
### Source code (`.fs`)  
  
| File | Change |  
|---|---|  
| `samples/GlobalRateLimiting/Program.fs:33` | `TimeSpan.FromSeconds(1)` → `TimeSpan.FromSeconds(1.0)` — .NET 10 added new `int64`-based overloads making an undecorated integer literal ambiguous (FS0041) |  
  
---  
  
## 5. ⏱️ Time Savings Estimate  
  
| Activity | Manual estimate | Automated |  
|---|---|---|  
| Research: F# multi-target → single TFM pattern | 2–3 h | ✅ Instant |  
| Identify NU1510 / platform-provided package removal | 1–2 h | ✅ Instant |  
| Diagnose NU1902 SourceLink transitive vulnerability + workaround | 2–4 h | ✅ Instant |  
| Discover and fix FS0041 `.NET 10` overload ambiguity in `TimeSpan.FromSeconds` | 1–3 h | ✅ Instant |  
| Update & validate all 7 project files | 1–2 h | ✅ Automated |  
| Full test run verification (454 tests) | 30 min | ✅ Automated |  
| **Total estimated saving** | **~8–14 hours** | **~4 minutes** |  
  
> 🚀 **Estimated 95%+ time reduction** versus manual migration by an experienced F# / .NET engineer.  
  
---  
  
## 6. ➡️ Next Steps  
  
### Validation  
- ✅ Run `dotnet build` on all 7 projects — confirmed: 0 errors, 0 warnings  
- ✅ Run `dotnet test tests/Giraffe.Tests/Giraffe.Tests.fsproj` — confirmed: 454 passed, 0 failed  
- 🔲 Run integration / smoke tests on each sample app against a live net10.0 runtime  
- 🔲 Verify NuGet package publish pipeline produces correct net10.0 `.nupkg` artefacts  
  
### Improvement Recommendations  
- 🔒 **Resolve GHSA-23fw-v26w-5fgq** (`Microsoft.Build.Tasks.Git` vulnerability): monitor the `Microsoft.SourceLink.GitHub` release feed for a patched version, then remove `NU1902` from `<NoWarn>` to restore full vulnerability auditing  
- 📦 **Consider re-enabling multi-targeting** (`net8.0;net9.0;net10.0`) on the published Giraffe NuGet package if downstream consumers are still on older runtimes — the library was architecturally multi-target before this upgrade  
- 🧹 **Review `FSharp.Core` pin**: library currently pins `FSharp.Core 6.0.0` with `DisableImplicitFSharpCoreReference`; this remains binary-compatible with net10.0 but may limit access to newer FSharp.Core APIs — consider raising the floor to `8.0.0` to align with the minimum supported runtime  
- 📊 **Analyzer compatibility**: `FSharp.Analyzers.Build 0.5.0`, `G-Research.FSharp.Analyzers 0.24.0`, and `Ionide.Analyzers 0.20.0` should be verified against net10.0 / F# 10 to ensure no analyzer diagnostics are suppressed silently  
- 🔁 **CI/CD pipeline**: update any pipeline matrix that runs tests against `net8.0` and `net9.0` to `net10.0` only, to match the updated project targets  

## Claude CLI usage

💳 Estimated Claude CLI usage (USD): 1.547876  
