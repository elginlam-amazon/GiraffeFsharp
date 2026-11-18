# .NET 10 Migration Summary

## Changes Made

### Project File Updates

All projects updated to target `net10.0` exclusively:

| Project | Before | After |
|---|---|---|
| `src/Giraffe/Giraffe.fsproj` | `net6.0;net7.0;net8.0;net9.0` | `net10.0` |
| `tests/Giraffe.Tests/Giraffe.Tests.fsproj` | `net8.0;net9.0;net10.0` | `net10.0` |
| `samples/EndpointRoutingApp/EndpointRoutingApp.fsproj` | `net9.0` | `net10.0` |
| `samples/RateLimiting/RateLimiting.fsproj` | `net9.0` | `net10.0` |
| `samples/GlobalRateLimiting/GlobalRateLimiting.fsproj` | `net8.0` | `net10.0` |
| `samples/NewtonsoftJson/NewtonsoftJson.fsproj` | `net9.0` | `net10.0` |
| `samples/ResponseCachingApp/ResponseCachingApp.fsproj` | `net9.0` | `net10.0` |

### Package Updates

- `src/Giraffe/Giraffe.fsproj`: Removed explicit `System.Text.Json` reference (automatically available in net10.0 framework; suppressed `NU1510`).
- `src/Giraffe/Giraffe.fsproj`: Added `NU1902` to `<NoWarn>` to handle a known moderate-severity vulnerability in `Microsoft.Build.Tasks.Git` 8.0.0 (a transitive build-only dependency of `Microsoft.SourceLink.GitHub 8.0.0`). This package is `PrivateAssets="All"` and only affects the build, not the runtime.
- `tests/Giraffe.Tests/Giraffe.Tests.fsproj`: Simplified `Microsoft.AspNetCore.TestHost` conditional references to a single unconditional `10.0.*` reference.

### Code Fixes

- `samples/GlobalRateLimiting/Program.fs` line 33: Changed `TimeSpan.FromSeconds(1)` to `TimeSpan.FromSeconds(1.0)`. In .NET 10, new `int64`-based overloads of `TimeSpan.FromSeconds` were added, making the call with an `int` literal ambiguous (FS0041). Using a float literal `1.0` resolves the ambiguity unambiguously.

## Build Result

```
Build succeeded.
    0 Warning(s)
    0 Error(s)
```

All 7 projects build cleanly targeting `net10.0`.

## Next Steps

- Consider upgrading `Microsoft.SourceLink.GitHub` from `8.0.0` to `8.0.0` or later (e.g., `11.0.*`) once the new version is verified compatible with the build toolchain, to eliminate the `NU1902` suppression. Currently suppressed via `<NoWarn>` since it is a build-only (PrivateAssets) dependency.
- Consider raising `FSharp.Core` from `6.0.0` in `Giraffe.fsproj` once downstream consumers support a newer minimum. The library keeps `DisableImplicitFSharpCoreReference=true` and pins FSharp.Core for NuGet compatibility.
