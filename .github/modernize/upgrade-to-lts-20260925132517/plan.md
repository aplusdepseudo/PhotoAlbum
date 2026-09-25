# .NET Upgrade Plan

## Overview

| Item | Value |
|------|-------|
| Application | PhotoAlbum |
| Language | dotnet |
| Solution | `PhotoAlbum.sln` |
| Source .NET version | **.NET 9.0** (`net9.0`) |
| Target .NET version | **.NET 10.0** (`net10.0`) — latest LTS |
| SDK-style conversion required | No (projects are already SDK-style) |

## Projects in Scope

| Project | Path | SDK | Current TFM |
|---------|------|-----|-------------|
| PhotoAlbum | `PhotoAlbum/PhotoAlbum.csproj` | `Microsoft.NET.Sdk.Web` | `net9.0` |
| PhotoAlbum.Tests | `PhotoAlbum.Tests/PhotoAlbum.Tests.csproj` | `Microsoft.NET.Sdk` | `net9.0` |

## Reason for Upgrade

The user explicitly requested an upgrade to .NET 10. .NET 9 is an STS release with a short support
window; moving to .NET 10 (LTS) provides long-term support, security servicing, performance
improvements, and access to the latest ASP.NET Core and EF Core features.

## What the Upgrade Entails

1. **Target framework update** — change `<TargetFramework>` from `net9.0` to `net10.0` in both projects.
2. **NuGet package updates** — upgrade dependencies to their .NET 10-compatible versions, notably:
   - `Microsoft.EntityFrameworkCore.SqlServer`, `Microsoft.EntityFrameworkCore.Design`,
     `Microsoft.EntityFrameworkCore.InMemory` (9.0.x → 10.0.x)
   - `Microsoft.AspNetCore.Mvc.Testing` (9.0.x → 10.0.x)
   - `Microsoft.NET.Test.Sdk`, `xunit`, `xunit.runner.visualstudio`, `coverlet.collector`
   - `SixLabors.ImageSharp` (verify latest compatible version)
3. **API / breaking-change remediation** — address any ASP.NET Core 10, EF Core 10, or BCL breaking
   changes surfaced during build (Razor Pages hosting, EF Core query/model APIs, analyzer warnings).
4. **Validation** — restore, build the full solution, and run the xUnit test suite in
   `PhotoAlbum.Tests` to confirm behavior is unchanged.

## Tasks

| # | Task ID | Type | Description |
|---|---------|------|-------------|
| 1 | `001-upgrade-dotnet-to-net10` | upgrade | Upgrade PhotoAlbum and PhotoAlbum.Tests from .NET 9.0 to .NET 10.0 |

## Success Criteria

- Solution builds successfully targeting `net10.0`.
- All existing unit tests pass.
