# .NET → `net10.0` transformation summary — Giraffe (F#)

## Result

- `dotnet build Giraffe.slnx -c Release`: **succeeded**, exit code 0, **0 errors, 0 warnings**.
- All seven requested projects build individually with exit code 0.
- `dotnet test tests/Giraffe.Tests` → **454 / 454 passed**, 0 failed, 0 skipped.
- `dotnet pack src/Giraffe` → produces `Giraffe.8.4.0.nupkg` with a single `lib/net10.0` leg.
- `dotnet fantomas --check src samples tests` → exit 0 (formatting clean).

## Target framework consolidation

| Project | Before | After |
| --- | --- | --- |
| `src/Giraffe/Giraffe.fsproj` | `net6.0;net7.0;net8.0;net9.0` | `net10.0` |
| `tests/Giraffe.Tests/Giraffe.Tests.fsproj` | `net8.0;net9.0;net10.0` | `net10.0` |
| `samples/EndpointRoutingApp` | `net9.0` | `net10.0` |
| `samples/NewtonsoftJson` | `net9.0` | `net10.0` |
| `samples/RateLimiting` | `net9.0` | `net10.0` |
| `samples/ResponseCachingApp` | `net9.0` | `net10.0` |
| `samples/GlobalRateLimiting` | `net8.0` | `net10.0` |

The plural `<TargetFrameworks>` form was kept on the two projects that already used it (per KB
`35-fsharp-migration.md`, step "Convert the project file"). `<Compile Include>` order was left
untouched in every `.fsproj`; no source file was added, removed or reordered.

## Package changes (`src/Giraffe/Giraffe.fsproj`)

- **Removed `System.Text.Json` 8.0.\*** — resolved `NU1510` (escalated to an error by the project's
  `TreatWarningsAsErrors=true`). `System.Text.Json` ships in the `net10.0` shared framework, so an
  explicit reference is neither needed nor prunable. A comment in the `.fsproj` records why.
- **Removed `Microsoft.SourceLink.GitHub` 8.0.\*** — resolved `NU1902` (moderate-severity advisory
  [GHSA-23fw-v26w-5fgq](https://github.com/advisories/GHSA-23fw-v26w-5fgq) against its transitive
  `Microsoft.Build.Tasks.Git` 8.0.0). SourceLink is built into the .NET SDK since .NET 8, and the
  project already sets `PublishRepositoryUrl` and `EmbedUntrackedSources`, so SourceLink behaviour
  is preserved by the SDK.
- **`Microsoft.AspNetCore.TestHost`** — the three per-TFM `ItemGroup`s (8.0.\* / 9.0.\* / 10.0.\*)
  collapsed to the single `net10.0` group at `10.0.*`.
- **`FSharp.Core` kept pinned at 6.0.0** with `DisableImplicitFSharpCoreReference`, as
  `35-fsharp-migration.md` step 3 prescribes for a library published as a NuGet package: raising the
  pin would raise the floor for every consumer, and no code here needs a newer FSharp.Core API.
- `Microsoft.IO.RecyclableMemoryStream`, `FSharp.SystemTextJson`, `Giraffe.ViewEngine`,
  `Newtonsoft.Json` (sample) are runtime-independent and were left at their current versions.

## Code changes

- `samples/GlobalRateLimiting/Program.fs:33` — `TimeSpan.FromSeconds(1)` → `TimeSpan.FromSeconds(1.0)`.
  .NET 9 added `TimeSpan.FromSeconds(int64)` and `FromSeconds(int64, …)` overloads, which made the
  `int` literal ambiguous for F# overload resolution (`FS0041`). The explicit `float` keeps the
  original 1-second fixed window. (`samples/RateLimiting` already used an explicit `int64 12`.)
- Collapsed conditional-compilation branches that became unconditional once only `net10.0` is built.
  Behaviour on `net10.0` is byte-for-byte identical; only the dead `#else` legs were dropped:
  - `src/Giraffe/Core.fs` and `src/Giraffe/HttpContextExtensions.fs` — removed `#if NET8_0_OR_GREATER`
    around the markdown content type, keeping `System.Net.Mime.MediaTypeNames.Text.Markdown`.
  - `tests/Giraffe.Tests/Helpers.fs` — removed `#if NET10_0_OR_GREATER` in `createHost` and
    `makeRequest`, keeping the `HostBuilder().ConfigureWebHost(… UseTestServer())` +
    `host.StartAsync()` / `host.GetTestServer()` form required from .NET 10
    ([aspnet/Announcements#526](https://github.com/aspnet/Announcements/issues/526)). The legacy
    `WebHostBuilder` / `new TestServer(...)` path is gone.

No test was deleted, skipped, disabled or weakened; no assertion was changed.

## CI

`.github/workflows/build-and-test.yml` and `.github/workflows/publish.yml` installed SDKs
`8.x`, `9.x`, `10.x` in four `actions/setup-dotnet` steps. With every project on `net10.0`, these
were trimmed to `10.x`. `fantomas-check.yml` pins no SDK and was left alone.

## Not applicable

The solution is already a modern ASP.NET Core / F# codebase. There is no `System.Web`,
`System.Data.Entity`, `System.Web.Http`, `Global.asax`, `Web.config`, EF6/Dapper/NHibernate, WCF,
`System.Drawing`, Swagger, Web Forms, SignalR 2, ASMX or .NET Remoting usage, so the pre-build
cleanup rules in `00-prebuild-cleanup-and-bulk-fixes.md` and KB files 02–34 had nothing to match.
Verified by grep across `src`, `tests` and `samples`.

## Next steps

Non-blocking items for human review in a later cycle:

1. **`NoWarn=FS2003;FS0044` in `src/Giraffe/Giraffe.fsproj` is still needed.** Building with the
   suppression lifted shows exactly two `FS0044` (deprecated-construct) warnings, both from
   Giraffe's *own* deliberate deprecation cycle rather than from the framework upgrade:
   - `src/Giraffe/Auth.fs:69` uses Giraffe's own `evaluateUserPolicy`, which is marked obsolete in
     favour of `authorizeUser`.
   - `src/Giraffe/HttpContextExtensions.fs:72` is the body of the obsolete
     `HttpContext.GetHostingEnvironment()` shim, which resolves ASP.NET Core's obsolete
     `IHostingEnvironment`. Giraffe's own internal call sites already use `GetWebHostEnvironment()` /
     `IWebHostEnvironment`.
   Clearing these means removing two public members, i.e. a breaking API change deferred to the
   "next major version" by their own `[<Obsolete>]` messages. Out of scope for a framework retarget —
   recommend doing it with the next major Giraffe release and then dropping `FS0044` from `NoWarn`.
2. **Dropping `System.Text.Json` from the package dependencies is a visible packaging change.**
   The published nuspec no longer lists `System.Text.Json`; consumers on `net10.0` get it from the
   shared framework. This is correct for a `net10.0`-only package, but it does mean a consumer that
   relied on Giraffe to float their `System.Text.Json` version now gets the framework-supplied one.
   Worth a line in `CHANGELOG.md` / `RELEASE_NOTES.md` when the next version is cut.
3. **The library no longer multi-targets.** Giraffe previously shipped `net6.0`–`net9.0` legs; the
   package now ships only `lib/net10.0`, so consumers on .NET 6–9 cannot use the next release. This
   follows directly from the requested transformation, but it is a consumer-facing decision that
   should be announced in the release notes (and the README's framework-support statement reviewed).
4. **F# analyzer tooling has a pre-existing version mismatch**, unrelated to this upgrade and outside
   `dotnet build`. `dotnet msbuild /t:AnalyzeFSharpProject src/Giraffe/Giraffe.fsproj` reports
   `Trying to load Ionide.Analyzers.dll which was built using SDK version 0.41.0.0. Expect 0.38.0.0`
   and then exits non-zero, because `.config/dotnet-tools.json` pins `fsharp-analyzers` 0.38.0 while
   `Ionide.Analyzers` 0.20.0 targets the 0.41 analyzers SDK. The CI analyzer job runs with
   `continue-on-error: true`, so this does not gate anything. Bumping `fsharp-analyzers` to 0.43.0
   may simply move the mismatch onto `G-Research.FSharp.Analyzers` 0.24.0, so the three versions
   should be aligned together rather than bumped piecemeal — left for a dedicated tooling pass.
   The run also surfaced four pre-existing style advisories (`GRA-STRING-002` ×3 in `ModelParser.fs`,
   `Core.fs`, `Routing.fs`; `GRA-TYPE-ANNOTATE-001` in `Routing.fs:12`) that predate this upgrade.
5. **Stale `bin/`/`obj/` directories owned by `root`** caused `MSB3374` ("last access/last write time
   cannot be set") on the very first build in this environment. They were world-writable but not
   owned by the build user, so MSBuild could not stamp timestamps. Deleting them resolved it; if a
   future cycle hits `MSB3374`, clear `bin`/`obj` first. No project change was needed.
6. `dotnet tool restore` notes newer `fantomas` (8.1.3) and `fsharp-analyzers` (0.43.0) are
   available. Left at the pinned versions deliberately — formatting already passes at 8.1.1, and
   bumping Fantomas can reflow unrelated files.
