# .NET 10 Migration Summary

## Migration target
All specified projects have been migrated to `net10.0`.

## Projects migrated

| Project | Before | After |
|---|---|---|
| `src/Giraffe/Giraffe.fsproj` | `net6.0;net7.0;net8.0;net9.0` (multi-target) | `net10.0` (single TFM) |
| `samples/EndpointRoutingApp/EndpointRoutingApp.fsproj` | `net9.0` | `net10.0` |
| `samples/RateLimiting/RateLimiting.fsproj` | `net9.0` | `net10.0` |
| `samples/GlobalRateLimiting/GlobalRateLimiting.fsproj` | `net8.0` | `net10.0` |
| `samples/NewtonsoftJson/NewtonsoftJson.fsproj` | `net9.0` | `net10.0` |
| `samples/ResponseCachingApp/ResponseCachingApp.fsproj` | `net9.0` | `net10.0` |
| `tests/Giraffe.Tests/Giraffe.Tests.fsproj` | `net8.0;net9.0;net10.0` (multi-target) | `net10.0` (single TFM) |

## Changes made

### src/Giraffe/Giraffe.fsproj
- Changed `<TargetFrameworks>net6.0;net7.0;net8.0;net9.0</TargetFrameworks>` → `<TargetFramework>net10.0</TargetFramework>` (single TFM; netstandard2.0 multi-targeting is excluded because the project references `Microsoft.AspNetCore.App` shared framework)
- Removed explicit `System.Text.Json` PackageReference — it is automatically available via the shared framework on net10.0 (NU1510)
- Updated `Microsoft.SourceLink.GitHub` from `8.0.*` to `10.0.*` to fix NU1902 vulnerability in `Microsoft.Build.Tasks.Git` 8.0.0
- `FSharp.Core` remains pinned at `6.0.0` (library practice: low explicit pin, binary-compatible with newer runtimes)

### tests/Giraffe.Tests/Giraffe.Tests.fsproj
- Collapsed `<TargetFrameworks>net8.0;net9.0;net10.0</TargetFrameworks>` → `<TargetFramework>net10.0</TargetFramework>`
- Removed per-TFM conditional `<ItemGroup>` blocks for `net8.0` and `net9.0`
- Left single `Microsoft.AspNetCore.TestHost` Version="10.0.*" reference

### samples/GlobalRateLimiting/Program.fs
- Fixed FS0041 overload ambiguity: `TimeSpan.FromSeconds(1)` → `TimeSpan.FromSeconds(1.0)`. .NET 10 added new `int64` overloads for `TimeSpan.FromSeconds`, making the unqualified integer literal `1` ambiguous between `int64` and `float` overloads. Using `1.0` selects the `float` overload unambiguously.

## Build results
All projects build with **0 errors, 0 warnings**:
- `src/Giraffe/Giraffe.fsproj` ✅
- `samples/EndpointRoutingApp/EndpointRoutingApp.fsproj` ✅
- `samples/RateLimiting/RateLimiting.fsproj` ✅
- `samples/GlobalRateLimiting/GlobalRateLimiting.fsproj` ✅
- `samples/NewtonsoftJson/NewtonsoftJson.fsproj` ✅
- `samples/ResponseCachingApp/ResponseCachingApp.fsproj` ✅
- `tests/Giraffe.Tests/Giraffe.Tests.fsproj` ✅

## Test results
454/454 tests pass on net10.0.

## Next steps
None. Migration is complete with zero errors and all tests passing.
