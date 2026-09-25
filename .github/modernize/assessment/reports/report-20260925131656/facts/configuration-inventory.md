# Configuration & Externalized Settings Inventory

PhotoAlbum uses a multi-layered configuration approach spanning local development (JSON-based appsettings), environment-specific overrides, User Secrets for sensitive local development values, and Infrastructure-as-Code (Bicep) for Azure cloud deployment. Configuration is primarily environment-driven, with ASPNETCORE_ENVIRONMENT selecting runtime behavior profiles and a test-environment flag controlling database migration execution.

## Configuration Sources

| Source | Type | Path/Location | Notes |
|--------|------|---------------|-------|
| appsettings.json | Application configuration | PhotoAlbum/appsettings.json | Base configuration for all environments (connection strings, file upload limits, logging defaults) |
| appsettings.Development.json | Environment override | PhotoAlbum/appsettings.Development.json | Development-specific overrides (detailed logging, error details) |
| launchSettings.json | Launch profiles | PhotoAlbum/Properties/launchSettings.json | Local development launch profiles (http, https) with ASPNETCORE_ENVIRONMENT and port bindings |
| .env.example | Environment template | .env.example | Template for Azure deployment environment variables (RESOURCE_GROUP) |
| azure.yaml | Azure Developer CLI | azure.yaml | Defines services, deployment configuration, infrastructure provider (Bicep), and hooks |
| User Secrets | Local secrets store | Project UserSecretsId: 28fdd5b1-4b72-4763-98cc-ac5ebb3f280d | Secure local storage for development secrets (connection strings, API keys) |
| main.bicep | Infrastructure template | infra/main.bicep | Bicep template for Azure resources (Container Apps, SQL Server, Container Registry, Log Analytics) |
| main.parameters.json | Template parameters | infra/main.parameters.json | Parameter values with environment variable references (AZURE_ENV_NAME, AZURE_LOCATION, SERVICE_WEB_IMAGE_NAME) |
| Dockerfile | Container configuration | Dockerfile | Multi-stage Docker build (base, build, publish, final) using .NET 9.0 SDK and runtime images |

## Build Profiles

| Profile | Activation | Purpose | Key Dependencies/Plugins |
|---------|-----------|---------|--------------------------|
| Debug | Default / Visual Studio / `dotnet build` | Debug compilation with symbols, no optimizations | All development packages |
| Release | `-c Release` flag or Visual Studio Release config | Optimized release build for production | All production packages with optimization |
| Any CPU | Default platform | Architecture-agnostic build (x64/x86/ARM) | Platform-independent .NET assemblies |
| x64 | `-p Platform=x64` (Visual Studio UI) | Explicit 64-bit architecture | 64-bit runtime-specific optimizations |
| x86 | `-p Platform=x86` (Visual Studio UI) | Explicit 32-bit architecture | 32-bit runtime support |

## Runtime Profiles

| Profile | Activation Method | Config Files | Key Overrides |
|---------|-------------------|--------------|----------------|
| Development | `ASPNETCORE_ENVIRONMENT=Development` (launchSettings.json default or environment variable) | appsettings.json + appsettings.Development.json | DetailedErrors: true; Logging levels: Debug (default), Information (AspNetCore, EF Core) |
| Production | Default (no explicit setting) | appsettings.json only | HSTS enabled; exception handler middleware; static file caching (3600 seconds) |
| Test | `IsTestEnvironment=true` configuration flag | appsettings.json with IsTestEnvironment override | Database migrations skipped; in-memory database support via test framework |

## Properties Inventory

### Core Application Properties

| Property Key | Default | Profiles | Type | Source |
|--------------|---------|----------|------|--------|
| ConnectionStrings:DefaultConnection | Server=(localdb)\\mssqllocaldb;Database=PhotoAlbumDb;Trusted_Connection=true;MultipleActiveResultSets=true | Development (local); Production (uses Bicep-generated connection string) | String | appsettings.json |
| FileUpload:MaxFileSizeBytes | 10485760 (10 MB) | All | Integer | appsettings.json |
| FileUpload:AllowedMimeTypes | ["image/jpeg", "image/png", "image/gif", "image/webp"] | All | String array | appsettings.json |
| FileUpload:MaxFilesPerUpload | 10 | All | Integer | appsettings.json |
| FileUpload:UploadPath | wwwroot/uploads | All | String | appsettings.json |
| Logging:LogLevel:Default | Information | All; Development overrides to Debug | String (enum: Trace, Debug, Information, Warning, Error, Critical, None) | appsettings.json |
| Logging:LogLevel:Microsoft.AspNetCore | Warning | All; Development overrides to Information | String | appsettings.json |
| Logging:LogLevel:Microsoft.EntityFrameworkCore | (not set in base) | Development sets to Information | String | appsettings.Development.json |
| Admin:Username | admin | All | String | appsettings.json |
| AllowedHosts | * | All | String (CORS) | appsettings.json |
| DetailedErrors | false (production) | Development sets to true | Boolean | appsettings.Development.json |
| IsTestEnvironment | false (default) | Test sets to true | Boolean | Programmatic configuration in tests |

### Azure Deployment Parameters (infra/main.parameters.json)

| Parameter Key | Default | Environment Variable | Type | Description |
|----------------|---------|----------------------|------|-------------|
| environmentName | (required) | ${AZURE_ENV_NAME} | String | Environment name (dev, test, prod) passed to Bicep |
| location | resourceGroup().location | ${AZURE_LOCATION} | String | Azure region for all resources |
| webImageName | mcr.microsoft.com/azuredocs/containerapps-helloworld:latest | ${SERVICE_WEB_IMAGE_NAME} | String | Container image URL (set by azd deploy) |

## Startup Parameters & Resource Requirements

| Component | Runtime Options | Memory | CPU | Instance Count | Notes |
|-----------|-----------------|--------|-----|-----------------|-------|
| PhotoAlbum Web Service (Local) | `dotnet run --project PhotoAlbum/PhotoAlbum.csproj` | Default .NET process | Default | 1 | Launches with ASPNETCORE_ENVIRONMENT from launchSettings.json |
| PhotoAlbum Web Service (Docker) | `dotnet PhotoAlbum.dll` | Container engine limits (Dockerfile specifies EXPOSE 8080) | Container engine limits | 1-N via Container Apps scaling | Multi-stage build (base: mcr.microsoft.com/dotnet/aspnet:9.0) |
| Database (LocalDB) | Integrated Windows authentication | Default SQL Server allocation | Default | 1 | Runs on local machine during development |
| Database (Azure SQL) | Azure AD Default authentication (managed identity) | Configured by Bicep (Basic tier: 2 GB max) | Configured by Bicep | 1 | Connection string: Server=tcp:{sqlServerFqdn},1433;Initial Catalog=PhotoAlbumDb;Authentication=Active Directory Default |
| Test Runner | `dotnet test PhotoAlbum.Tests/PhotoAlbum.Tests.csproj` | Default .NET process | Default | 1 | Uses in-memory database; sets IsTestEnvironment=true |

## Startup Dependency Chain

1. **Local Development:**
   - Application starts → Uploads directory creation (if missing) → Database migration (auto-apply) → Service registration → HTTP middleware pipeline → Ready for requests
   - Dependency: appsettings (load) → connection string validation → database connectivity

2. **Azure Container Apps Deployment:**
   - Infrastructure provision (Bicep: Log Analytics → Container Apps Environment → Container Registry → SQL Server + Database) → Post-provision hook (wait 60-120s for role propagation) → Image push to ACR → Pre-deploy hook (configure managed identity) → Container App deployment → Ready

3. **Database Connectivity:**
   - Application reads ConnectionStrings:DefaultConnection → connects to SQL Server → runs migrations (unless IsTestEnvironment=true) → DbContext ready
   - Wait mechanism: Synchronous; migration failure causes app startup failure

4. **File Upload Readiness:**
   - Uploads directory created on startup → wwwroot/uploads path accessible → FormOptions configured (MultipartBodyLengthLimit: 10MB)

5. **Authentication/Authorization:**
   - Cookie authentication scheme configured → Login page mapped → Authorization middleware ready for photo deletion operations

## Secrets & Sensitive Configuration

| Secret Reference | Type | Storage | Provisioning Source |
|------------------|------|---------|-------------------|
| ConnectionStrings:DefaultConnection (Azure) | Database connection string with credentials | [MASKED] - Azure Key Vault or managed identity | Bicep template (variables:sqlConnectionString; uses Active Directory Default auth) |
| SQL Admin Password | SQL Server admin credential | [MASKED] - Generated locally in Bicep | Bicep: `P@ssw0rd${uniqueString(resourceGroup().id, environmentName)}` |
| SQL Admin User | SQL Server admin username | sqladmin | Bicep (hardcoded for basic deployments) |
| User Secrets (local development) | Development-only connection strings, API keys | ~/.microsoft/usersecrets/28fdd5b1-4b72-4763-98cc-ac5ebb3f280d/ | User Secrets CLI (`dotnet user-secrets set`) |
| Azure Container Registry Admin Credentials | ACR login credentials | [MASKED] - Azure ACR (basic SKU) | Bicep: `acrAdminUserEnabled: true` |
| Azure Managed Identity | System-assigned identity for Container App | Azure Managed Identity | Bicep: implicit in Container App resource |

### Secrets Provisioning Workflow

**Development Environment:**
- Secret source: User Secrets store (local machine) or appsettings.Development.json
- Identity/access model: Windows authenticated login to LocalDB (Trusted_Connection=true)
- Provisioning sequence: Developer runs `dotnet user-secrets set "ConnectionStrings:DefaultConnection" "<conn_string>"` → ASP.NET reads from secrets store → Database connection established
- Services needing secrets: PhotoAlbum web service (SQL connection, optional API keys)

**Azure Production Deployment:**
- Secret source: Bicep template generates SQL admin password; Azure SQL Server uses Active Directory Default authentication (managed identity)
- Identity/access model: System-assigned managed identity on Container App with implicit Azure AD token; no explicit passwords stored in application configuration
- Provisioning sequence:
  1. `azd provision` → Bicep deploys SQL Server with managed identity-enabled database
  2. Container App resource created with system-assigned managed identity
  3. Post-provision hook waits for role assignments to propagate (60-120 seconds)
  4. Pre-deploy hook configures Container App ACR registry with managed identity (system identity)
  5. `azd deploy` → Container image pushed to ACR; Container App pulls using managed identity (AcrPull role)
  6. Container App connects to SQL using Active Directory Default auth (token-based, no password)
- Which services need which secrets: PhotoAlbum Container App needs SQL database connection (via managed identity); no explicit credentials in appsettings.json for Azure

**Local Docker Testing:**
- Environment variables passed via docker-compose or `docker run -e` flags
- Secrets can be injected via Docker secrets or mounted volumes (not currently configured)

## Feature Flags

| Flag Name | Default | Controlled By | Purpose |
|-----------|---------|---------------|---------|
| IsTestEnvironment | false | Programmatic configuration in test setup or environment variable | Controls whether database migrations run on startup (skipped when true for in-memory test databases) |
| ASPNETCORE_ENVIRONMENT | Development (launchSettings.json) | Environment variable or launchSettings.json profileName | Determines which appsettings.{Environment}.json overrides are loaded; controls error handling, logging, HSTS, static file caching |
| app.Environment.IsDevelopment() | Depends on ASPNETCORE_ENVIRONMENT | .NET runtime check | Middleware conditional: UseExceptionHandler, UseHsts, detailed logging in development |

## Framework & Runtime Versions

| Component | Version | Source |
|-----------|---------|--------|
| .NET Runtime Target | 9.0 | PhotoAlbum.csproj: `<TargetFramework>net9.0</TargetFramework>` |
| ASP.NET Core | 9.0 (implicit in .NET 9.0) | .NET 9.0 SDK |
| Entity Framework Core | 9.0.9 | PhotoAlbum.csproj PackageReference |
| Entity Framework Core SQL Server | 9.0.9 | PhotoAlbum.csproj PackageReference |
| SixLabors.ImageSharp | 3.1.11 | PhotoAlbum.csproj PackageReference |
| .NET Test SDK | 17.12.0 | PhotoAlbum.Tests.csproj PackageReference |
| xUnit | 2.9.2 | PhotoAlbum.Tests.csproj PackageReference |
| xUnit Runner VisualStudio | 2.8.2 | PhotoAlbum.Tests.csproj PackageReference |
| coverlet.collector | 6.0.2 | PhotoAlbum.Tests.csproj PackageReference (code coverage) |
| Microsoft.AspNetCore.Mvc.Testing | 9.0.9 | PhotoAlbum.Tests.csproj PackageReference (integration test framework) |
| Entity Framework Core In-Memory | 9.0.9 | PhotoAlbum.Tests.csproj PackageReference (test database provider) |
| Docker Base Image (Runtime) | mcr.microsoft.com/dotnet/aspnet:9.0 | Dockerfile: base stage |
| Docker Base Image (Build) | mcr.microsoft.com/dotnet/sdk:9.0 | Dockerfile: build stage |
| Build Tool | .NET CLI / MSBuild (implicit) | Visual Studio 2022 or .NET 9.0 SDK command line |
| Azure Bicep Version | 0.38.3.11034 | infra/main.json metadata (generated from bicep CLI) |

## Additional Configuration Notes

- **Nullable Reference Types**: Enabled in PhotoAlbum.csproj (`<Nullable>enable</Nullable>`)
- **Implicit Usings**: Enabled in both PhotoAlbum.csproj and PhotoAlbum.Tests.csproj
- **Static Assets**: Configured with 1-hour cache control in Program.cs (Cache-Control: public,max-age=3600)
- **File Upload Limits**: MultipartBodyLengthLimit and ValueLengthLimit both set to 10MB (10485760 bytes) in Program.cs FormOptions
- **Form Options Boundary**: MultipartBoundaryLengthLimit set to 128 bytes (default)
- **Azure Developer CLI Integration**: azure.yaml defines service language as "dotnet", host as "containerapp", and orchestration via Bicep IaC
