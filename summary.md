# .NET Migration Summary

## Migration: net6.0/net7.0/net8.0/net9.0 → net10.0

### Projects migrated

| Project | Before | After |
|---|---|---|
| `src/Giraffe/Giraffe.fsproj` | `net6.0;net7.0;net8.0;net9.0` | `net10.0` |
| `tests/Giraffe.Tests/Giraffe.Tests.fsproj` | `net8.0;net9.0;net10.0` | `net10.0` |
| `samples/EndpointRoutingApp/EndpointRoutingApp.fsproj` | `net9.0` | `net10.0` |
| `samples/RateLimiting/RateLimiting.fsproj` | `net9.0` | `net10.0` |
| `samples/GlobalRateLimiting/GlobalRateLimiting.fsproj` | `net8.0` | `net10.0` |
| `samples/NewtonsoftJson/NewtonsoftJson.fsproj` | `net9.0` | `net10.0` |
| `samples/ResponseCachingApp/ResponseCachingApp.fsproj` | `net9.0` | `net10.0` |

### Changes made

#### `src/Giraffe/Giraffe.fsproj`
- Changed `<TargetFrameworks>net6.0;net7.0;net8.0;net9.0</TargetFrameworks>` to `<TargetFrameworks>net10.0</TargetFrameworks>`
- Removed explicit `System.Text.Json` PackageReference (NU1510: automatically available via `Microsoft.AspNetCore.App` FrameworkReference on net10.0)
- Updated `Microsoft.SourceLink.GitHub` from `8.0.*` to `9.0.*`
- Added `NU1902` to `<NoWarn>` to suppress the `Microsoft.Build.Tasks.Git` vulnerability warning (all current versions of this SourceLink transitive dependency are reported as affected; it is a build-time-only tool with `PrivateAssets="All"` so does not affect library consumers)

#### `tests/Giraffe.Tests/Giraffe.Tests.fsproj`
- Changed `<TargetFrameworks>net8.0;net9.0;net10.0</TargetFrameworks>` to `<TargetFrameworks>net10.0</TargetFrameworks>`
- Collapsed three conditional `Microsoft.AspNetCore.TestHost` ItemGroups (net8.0, net9.0, net10.0) into one unconditional reference at `Version="10.0.*"`

#### `samples/GlobalRateLimiting/Program.fs`
- Fixed `TimeSpan.FromSeconds(1)` → `TimeSpan.FromSeconds(1.0)`: .NET 10 added new `int64`-based overloads making the `int` literal ambiguous (FS0041)

#### All sample projects
- Updated `<TargetFramework>` to `net10.0`

### Build result
All 7 projects build with **0 warnings, 0 errors**.

### Test result
454/454 tests pass on net10.0.

### Next steps
- `NU1902` suppression for `Microsoft.Build.Tasks.Git`: monitor for a fixed version of `Microsoft.SourceLink.GitHub` that resolves GHSA-23fw-v26w-5fgq, then remove `NU1902` from `<NoWarn>` and restore normal vulnerability scanning.
