# .NET Migration Summary: net6.0/net7.0/net8.0/net9.0 → net10.0

## Changes made

### src/Giraffe/Giraffe.fsproj
- Changed `TargetFrameworks` from `net6.0;net7.0;net8.0;net9.0` to `net10.0`
- Removed explicit `PackageReference` for `System.Text.Json` (now part of the shared framework on net10.0, NU1510 raised it as an error)
- Updated `Microsoft.SourceLink.GitHub` from `8.0.*` to `9.0.*`
- Added `NU1902` to `NoWarn`: all currently published versions of `Microsoft.Build.Tasks.Git` carry advisory GHSA-23fw-v26w-5fgq; this is a build-time-only package (`PrivateAssets="All"`) with no runtime exposure

### samples/EndpointRoutingApp/EndpointRoutingApp.fsproj
- Changed `TargetFramework` from `net9.0` to `net10.0`

### samples/RateLimiting/RateLimiting.fsproj
- Changed `TargetFramework` from `net9.0` to `net10.0`

### samples/GlobalRateLimiting/GlobalRateLimiting.fsproj
- Changed `TargetFramework` from `net8.0` to `net10.0`

### samples/GlobalRateLimiting/Program.fs
- Fixed FS0041 ambiguous overload: `TimeSpan.FromSeconds(1)` → `TimeSpan.FromSeconds(1.0)`. .NET 10 added new `int64` overloads of `TimeSpan.FromSeconds` that made an `int` literal ambiguous for the F# overload resolver; using a `float` literal selects the existing `float` overload unambiguously.

### samples/NewtonsoftJson/NewtonsoftJson.fsproj
- Changed `TargetFramework` from `net9.0` to `net10.0`

### samples/ResponseCachingApp/ResponseCachingApp.fsproj
- Changed `TargetFramework` from `net9.0` to `net10.0`

### tests/Giraffe.Tests/Giraffe.Tests.fsproj
- Changed `TargetFrameworks` from `net8.0;net9.0;net10.0` to `net10.0`
- Collapsed three per-framework conditional `Microsoft.AspNetCore.TestHost` references into one unconditional `Version="10.0.*"` reference

## Build result
All 7 projects build with zero errors and zero warnings on .NET 10.0.404 SDK.

## Next steps
- **NU1902 (GHSA-23fw-v26w-5fgq)**: Advisory on `Microsoft.Build.Tasks.Git` affects all currently published versions. Monitor https://github.com/advisories/GHSA-23fw-v26w-5fgq for a patched release; once available, update `Microsoft.SourceLink.GitHub` to that version and remove `NU1902` from `NoWarn`.
- `FSharp.Core` is pinned at `6.0.0` in the library (intentional — library keeps the lowest compatible version as a NuGet dependency floor). No change needed unless a newer FSharp.Core API is required.
