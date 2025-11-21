# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build, Test, and Run Commands

### Build
```bash
dotnet restore CleanArchitecture.Blazor.slnx
dotnet build CleanArchitecture.Blazor.slnx --configuration Debug
```

### Run Application
```bash
dotnet run --project src/Server.UI
```
Access at `https://localhost:7152`

### Database Migrations
```bash
# MSSQL
dotnet ef database update --project src/Migrators/Migrators.MSSQL

# PostgreSQL
dotnet ef database update --project src/Migrators/Migrators.PostgreSQL

# SQLite
dotnet ef database update --project src/Migrators/Migrators.SqLite
```

### Testing
```bash
# Run all tests
dotnet test

# Run specific test project
dotnet test tests/Application.UnitTests
dotnet test tests/Application.IntegrationTests
dotnet test tests/Domain.UnitTests
dotnet test tests/Infrastructure.UnitTests
```

## Architecture Overview

### Clean Architecture Layers

This is a Blazor Server application built with Clean Architecture principles and strict dependency flow:

```
Domain (Core) ← Application ← Infrastructure
                      ↑              ↑
                      └── Server.UI ─┘
```

- **Domain** (`src/Domain`): Core entities, domain events, and business rules. No external dependencies.
- **Application** (`src/Application`): Business logic via MediatR CQRS pattern, validators, DTOs, specifications, and pipeline behaviors.
- **Infrastructure** (`src/Infrastructure`): Data access (EF Core), external services (email, files, PDF, OCR), and multi-tenancy.
- **Server.UI** (`src/Server.UI`): Blazor Server components, pages, SignalR hubs, and UI services.
- **Migrators** (`src/Migrators/*`): Provider-specific EF Core migrations for SQLite, MSSQL, and PostgreSQL.

### Key Patterns

**CQRS with MediatR**: All business operations are implemented as commands (write) or queries (read) handled through MediatR.

**Database Access Pattern**: Always use `IApplicationDbContextFactory` instead of injecting `ApplicationDbContext` directly:
```csharp
private readonly IApplicationDbContextFactory _dbContextFactory;

public async Task<Result> Handle(Command request, CancellationToken ct)
{
    await using var db = await _dbContextFactory.CreateAsync(ct);
    // Use db...
}
```
This ensures proper tenant scoping, lifetime management, and prevents concurrency issues in Blazor Server's long-lived circuits.

**Caching**: Integrated at MediatR pipeline level using FusionCache. Queries implement `ICacheableRequest`, commands implement `ICacheInvalidatorRequest`. Each feature has a `CacheKey` class with tags for invalidation.

**Specifications**: Ardalis.Specification pattern for composable query logic. Each entity has specifications like `ByIdSpecification` and `AdvancedSpecification` for filtering.

**Domain Events**: Entities raise events (Created/Updated/Deleted) handled by event handlers in the Application layer.

**Validation**: FluentValidation integrated via MediatR pipeline behavior. Each command has a corresponding validator.

### Multi-Tenancy

Multi-tenant architecture with tenant isolation at the database level:
- Claims principal factory enriches user identity with tenant context
- `TenantDataSourceService` and tenant switching capability
- Tenant scoping handled by DbContext factory pattern

### Security & Identity

- ASP.NET Core Identity with custom `AuditSignInManager` for login tracking
- Risk analysis via `SecurityAnalysisHeuristics` detecting brute-force, unusual times, new devices/locations
- Permission-based authorization with granular permissions per feature (View, Create, Edit, Delete, Export, etc.)
- Permissions auto-seeded from `Permissions` class nested static classes

## Adding a New Entity/Feature

Follow the Contacts pattern as the reference implementation. The complete guide is in `openspec/project.md` section "New Entity/Feature Guide (Contacts Pattern)".

### Quick Summary

1. **Domain Layer** (`src/Domain`):
   - Create entity class deriving from `BaseAuditableEntity`
   - Add domain events (Created/Updated/Deleted) deriving from `DomainEvent`

2. **Infrastructure Layer** (`src/Infrastructure`):
   - Add EF configuration implementing `IEntityTypeConfiguration<TEntity>`
   - Add seed data in `ApplicationDbContextInitializer.SeedDataAsync()`

3. **Application Layer** (`src/Application/Features/<Entities>`):
   - **Caching/**: Create `<Entity>CacheKey.cs` with tags and refresh logic
   - **Commands/**: Create/Update/Delete commands implementing `ICacheInvalidatorRequest`
   - **DTOs/**: Create DTO with AutoMapper profile
   - **EventHandlers/**: Add handlers for domain events
   - **Queries/**: GetById, GetAll, Pagination, Export queries
   - **Security/**: Add permissions class to `Permissions` with View/Create/Edit/Delete/etc.
   - **Specifications/**: ById and Advanced specifications

4. **UI Layer** (`src/Server.UI/Pages/<Entities>`):
   - Index/list page with `MudDataGrid`
   - Create/Edit/View pages
   - Dialog component for forms
   - Add menu entry in `src/Server.UI/Services/Navigation/MenuService.cs`

**Reference Files**:
- Entity: `src/Domain/Entities/Contact.cs:8`
- Configuration: `src/Infrastructure/Persistence/Configurations/ContactConfiguration.cs:9`
- Cache: `src/Application/Features/Contacts/Caching/ContactCacheKey.cs:1`
- Command: `src/Application/Features/Contacts/Commands/AddEdit/AddEditContactCommand.cs:1`
- Query: `src/Application/Features/Contacts/Queries/Pagination/ContactsPaginationQuery.cs:1`
- Page: `src/Server.UI/Pages/Contacts/Contacts.razor:1`

## OpenSpec Workflow

This project uses OpenSpec for spec-driven development with formal change proposals. See `openspec/AGENTS.md` for complete instructions.

### Key Commands
```bash
# List specifications and changes
openspec list --specs
openspec list

# Show details
openspec show <change-id>
openspec show <spec-id> --type spec

# Validate changes
openspec validate <change-id> --strict

# Archive after deployment
openspec archive <change-id> --yes
```

### When to Create a Proposal

Create a proposal (`openspec/changes/<change-id>/`) for:
- New features or capabilities
- Breaking changes (API, schema)
- Architecture or pattern changes
- Security updates

Skip proposal for:
- Bug fixes restoring spec behavior
- Typos, formatting, comments
- Non-breaking dependency updates

### Proposal Structure

Each change requires:
- `proposal.md`: Why, what changes, impact
- `tasks.md`: Implementation checklist
- `design.md`: Technical decisions (only when needed for cross-cutting changes, new dependencies, or architectural patterns)
- `specs/<capability>/spec.md`: Spec deltas with `## ADDED|MODIFIED|REMOVED Requirements`

Each requirement MUST have at least one `#### Scenario:` (4 hashtags). Always validate with `openspec validate <id> --strict` before requesting approval.

See `openspec/project.md` for detailed conventions and the Contacts pattern reference.

## Configuration

### Database Provider

Configure in `appsettings.json`:
```json
{
  "DatabaseSettings": {
    "DBProvider": "mssql|postgresql|sqlite",
    "ConnectionString": "..."
  }
}
```

### External Services

- **MinIO** (Object Storage): `Minio` section with `Endpoint`, `AccessKey`, `SecretKey`, `BucketName`
- **SMTP** (Email): `SmtpClientOptions` with `Server`, `Port`, `User`, `Password`
- **Gemini API** (OCR): `AI:GeminiApiKey`
- **Serilog** (Logging): `Serilog:WriteTo` array with sinks (Console, SQLite, PostgreSQL, MSSqlServer, Seq)
- **Hangfire**: Background jobs, dashboard at `/jobs`
- **OAuth Providers**: `Authentication` section for Microsoft, Google, Facebook

## Important Constraints

- Target framework: `net9.0` (NOTE: CI workflow shows .NET 10 in `dotnet.yml:26` but projects may be on .NET 9 - verify project files)
- Database access: Always use `IApplicationDbContextFactory` pattern, never inject `ApplicationDbContext` directly
- Caching: Use `<Entity>CacheKey.Tags` consistently for invalidation
- Permissions: Auto-discovered from `Permissions` nested classes, no manual registration needed
- Migrations: Update all provider-specific migrators when changing schema
- SignalR: Max receive message size 64 KB
- Hangfire: Default storage is in-memory (configure persistent storage for production)
