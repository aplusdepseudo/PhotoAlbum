# Architecture Diagram

PhotoAlbum is an ASP.NET Core 9.0 Razor Pages application that manages a photo gallery with file upload, storage, and retrieval capabilities. The application demonstrates modern cloud-ready patterns with service layer abstraction, transactional consistency, and configuration-driven deployment.

## Application Architecture

```mermaid
flowchart TD
    subgraph Client["Client Layer"]
        Browser["Web Browser"]
    end
    subgraph Presentation["Presentation Layer - ASP.NET Core 9.0 Razor Pages"]
        IndexPage["Index.cshtml + IndexModel"]
        DetailPage["Detail.cshtml + DetailModel"]
        LoginPage["Login.cshtml + LoginModel"]
        PhotoFilePage["PhotoFile.cshtml + PhotoFileModel"]
        ErrorPage["Error.cshtml"]
    end
    subgraph Business["Business Logic Layer"]
        PhotoService["PhotoService - IPhotoService"]
        ImageSharp["ImageSharp 3.1.11<br/>Image Processing"]
        Auth["Cookie Authentication"]
    end
    subgraph DataAccess["Data Access Layer - EF Core 9.0"]
        DbContext["PhotoAlbumContext"]
        PhotoModel["Photo Entity Model"]
    end
    subgraph Storage["Data Storage"]
        DB[("SQL Server LocalDB<br/>PhotoAlbumDb")]
        FileSystem["File System<br/>wwwroot/uploads"]
    end
    subgraph Config["Configuration"]
        AppSettings["appsettings.json<br/>Connection Strings & File Rules"]
    end

    Browser -->|"HTTP requests"| IndexPage
    Browser -->|"HTTP requests"| DetailPage
    Browser -->|"HTTP requests"| LoginPage
    Browser -->|"Static files"| PhotoFilePage
    
    IndexPage -->|"OnGet/OnPost"| PhotoService
    DetailPage -->|"OnGet/OnPost"| PhotoService
    LoginPage -->|"OnPost"| Auth
    PhotoFilePage -->|"OnGet"| PhotoService
    ErrorPage -->|"Error handling"| Presentation
    
    PhotoService -->|"Process images<br/>Extract dimensions"| ImageSharp
    PhotoService -->|"CRUD operations"| DbContext
    Auth -->|"Validates credentials<br/>Issues cookies"| PhotoService
    
    DbContext -->|"Map entities"| PhotoModel
    DbContext -->|"SQL queries"| DB
    PhotoService -->|"Save files"| FileSystem
    
    PhotoService -->|"Read config"| AppSettings
    Auth -->|"Read config"| AppSettings
```

### Technology Stack Summary

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Presentation | ASP.NET Core Razor Pages | 9.0 | Server-side web framework for page rendering and form handling |
| Presentation | C# | 12.0 (implicit) | Primary programming language |
| Business Logic | PhotoService | Custom | Service layer abstracting photo operations and file handling |
| Business Logic | SixLabors.ImageSharp | 3.1.11 | Image processing library for dimension extraction and format validation |
| Authentication | ASP.NET Core Identity (Cookies) | 9.0 | Cookie-based authentication for admin operations |
| Data Access | Entity Framework Core | 9.0 | ORM for database operations and migrations |
| Data Storage | SQL Server LocalDB | 2022+ | Local development database with automatic migration support |
| File Storage | Windows File System | N/A | GUID-based photo file storage in wwwroot/uploads |
| Configuration | appsettings.json | N/A | Externalized configuration for connection strings and file upload rules |

### Data Storage & External Services

The application uses **SQL Server LocalDB** as the primary data store, managing photo metadata including original filename, stored filename, file size, MIME type, image dimensions, and upload timestamps. Photos are stored on the **local file system** in `wwwroot/uploads/` with GUID-based filenames to prevent collisions and security issues. Configuration is externalized through `appsettings.json`, enabling easy customization of database connection strings, file size limits (10MB default), allowed MIME types (JPEG, PNG, GIF, WebP), and upload paths. The application is designed for migration to **Azure Blob Storage** as evidenced by the service layer abstraction through `IPhotoService`, allowing transparent swapping of storage implementations.

### Key Architectural Decisions

- **Service Layer Abstraction**: `IPhotoService` interface decouples business logic from storage implementation, enabling future migration from local file storage to Azure Blob Storage without affecting page models or request handlers.
- **Transactional Consistency**: File deletion on database save failure prevents orphaned files, ensuring data consistency between the database and file system.
- **Configuration-Driven Deployment**: File size limits, allowed MIME types, upload paths, and database connection strings are externalized in `appsettings.json`, enabling environment-specific configurations without code changes.

## Component Relationships

```mermaid
flowchart LR
    subgraph Presentation["Presentation Layer"]
        IndexModel["IndexModel<br/>GET: Load photos<br/>POST: Upload files"]
        DetailModel["DetailModel<br/>GET: Display photo<br/>POST: Delete photo"]
        LoginModel["LoginModel<br/>POST: Authenticate user"]
        PhotoFileModel["PhotoFileModel<br/>GET: Retrieve file"]
        ErrorModel["ErrorModel<br/>Error handling"]
    end
    subgraph Business["Business Logic Layer"]
        PhotoService["PhotoService<br/>IPhotoService"]
        AuthService["Authentication Service<br/>Cookie-based auth"]
    end
    subgraph DataAccess["Data Access Layer"]
        DbContext["PhotoAlbumContext<br/>DbContext"]
        PhotoEntity["Photo Entity<br/>Model class"]
    end
    subgraph Infrastructure["Infrastructure & External"]
        ImageLib["ImageSharp<br/>Image operations"]
        FileIO["File I/O<br/>Read/Write files"]
        DB[("SQL Server LocalDB<br/>Database")]
    end

    IndexModel -->|"GetAllPhotosAsync()<br/>UploadPhotoAsync()"| PhotoService
    DetailModel -->|"GetAllPhotosAsync()<br/>DeletePhotoAsync()"| PhotoService
    LoginModel -->|"Authenticate"| AuthService
    PhotoFileModel -->|"GetPhotoFileAsync()"| PhotoService
    ErrorModel -.->|"Error handling"| Presentation
    
    PhotoService -->|"Validate file<br/>Extract dimensions"| ImageLib
    PhotoService -->|"CRUD operations"| DbContext
    AuthService -->|"Cookie validation"| PhotoService
    
    DbContext -->|"Query/Create/Update/Delete"| PhotoEntity
    DbContext -->|"Execute migrations<br/>SQL queries"| DB
    PhotoService -->|"Save/Delete files"| FileIO
    ImageLib -->|"Process images"| FileIO
```

### Component Inventory

| Component | Layer | Type | Responsibility |
|-----------|-------|------|-----------------|
| IndexModel | Presentation | Razor Page Model | Handles gallery view (GET) and multi-file upload (POST) requests; delegates to PhotoService for data operations |
| DetailModel | Presentation | Razor Page Model | Displays single photo in full size (GET) with metadata and navigation; handles authenticated photo deletion (POST) |
| LoginModel | Presentation | Razor Page Model | Processes login form submissions and establishes authenticated sessions |
| PhotoFileModel | Presentation | Razor Page Model | Retrieves photo files by ID for direct download or display |
| ErrorModel | Presentation | Razor Page Model | Centralized error page and global exception handling |
| PhotoService | Business Logic | Service | Core business logic for photo upload validation, file processing, database persistence, and deletion; abstracts storage implementation |
| Authentication Service | Infrastructure | Middleware | Manages cookie-based authentication, validates user credentials, protects admin operations like photo deletion |
| PhotoAlbumContext | Data Access | EF Core DbContext | Maps Photo entity to database tables; manages database schema, migrations, and connection lifecycle |
| Photo Entity | Data Access | Entity Model | Represents photo metadata (original filename, stored filename, dimensions, MIME type, file size, upload timestamp) |
| ImageSharp Library | Infrastructure | External Library | Processes uploaded images to extract dimensions and validate format before storage |
| File I/O | Infrastructure | System | Handles physical file operations (save, retrieve, delete) on the file system in wwwroot/uploads/ |
| SQL Server LocalDB | Data Storage | Database | Stores photo metadata with descending index on UploadedAt for efficient chronological queries |
