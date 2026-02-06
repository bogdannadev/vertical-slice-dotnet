# BonusSystem — Vertical Slice Architecture Reference Implementation

> A production-style **bonus loyalty platform** built to demonstrate **Vertical Slice Architecture (VSA)** with the **Backend for Frontend (BFF)** pattern. Think of it as the [eShopOnContainers](https://github.com/dotnet/eShop) for vertical slices — a working, multi-role system you can study, fork, and adapt.

**Stack:** .NET 9 · PostgreSQL 17 · Docker Compose · JWT · Swagger/OpenAPI

---

## What Is Vertical Slice Architecture?

Vertical Slice Architecture (VSA) organises code by **feature** rather than **technical layer**. Instead of scattering a single feature across Controllers, Services, and Repositories (the traditional N-tier approach), VSA keeps everything a feature needs in one place — its own *slice*.

The term was popularised by [Jimmy Bogard](https://www.jimmybogard.com/vertical-slice-architecture/) and draws on ideas from **Feature-Driven Development**, **CQRS**, and the principle that *code that changes together should live together*.

### The Problem VSA Solves

In a traditional layered architecture, adding a single feature — say, "let buyers cancel transactions" — forces you to touch multiple layers:

```
Traditional N-Tier (horizontal slicing)
────────────────────────────────────────

  ┌─────────────────────────────────────────────┐
  │            Controller Layer                 │  ← modify UserController
  ├─────────────────────────────────────────────┤
  │            Service Layer                    │  ← modify UserService
  ├─────────────────────────────────────────────┤
  │            Repository Layer                 │  ← modify UserRepository
  └─────────────────────────────────────────────┘

  Problem: a buyer-only feature ripples through layers
  shared with admin, seller, and observer logic.
```

Every layer becomes a shared namespace. Over time, services accumulate role-specific branches (`if (role == Admin) ...`), DTOs bloat with fields irrelevant to most consumers, and changing one role risks breaking another.

### How VSA Fixes This

VSA slices the application *vertically* — each feature owns its endpoints, request/response models, and business logic:

```
Vertical Slice Architecture
────────────────────────────

  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  Admin   │  │  Buyer   │  │  Seller  │  │ Observer │
  │  Slice   │  │  Slice   │  │  Slice   │  │  Slice   │
  │          │  │          │  │          │  │          │
  │ Endpoints│  │ Endpoints│  │ Endpoints│  │ Endpoints│
  │ BFF Svc  │  │ BFF Svc  │  │ BFF Svc  │  │ BFF Svc  │
  │ Requests │  │ Requests │  │ Requests │  │ Requests │
  │ Responses│  │ Responses│  │ Responses│  │ Responses│
  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │             │             │             │
       └─────────────┴──────┬──────┴─────────────┘
                            │
                    Shared Repository
                      Contracts Only
                            │
                        PostgreSQL
```

Changes to the buyer slice never touch admin code. Each slice is independently testable, deployable, and comprehensible.

### Trade-offs

| Advantage | Disadvantage |
|-----------|--------------|
| Changes stay within one slice | Some code duplication across slices (by design) |
| Clearer role-based boundaries | Requires discipline to avoid cross-slice coupling |
| Easier onboarding — understand one slice at a time | Shared concerns (auth, logging) need careful abstraction |
| Natural path to microservices if needed | Teams unfamiliar with VSA may default to layered habits |
| Simpler testing — mock only what the slice needs | Pattern less documented than N-tier or clean architecture |

### When to Use VSA

VSA shines in systems with **multiple user roles**, **distinct feature boundaries**, or **teams that own features end-to-end**. It's less beneficial for CRUD-heavy applications with uniform access patterns where traditional layers suffice.

---

## Why This Repository?

This project exists to fill a gap: most VSA discussions stay at the blog-post level. There are few complete, working implementations that show how the pattern plays out in a real multi-role system with actual business logic.

**BonusSystem serves as:**

| Audience | Value |
|----------|-------|
| Junior developers | A hands-on guide to VSA with production-style code to read and study |
| Senior developers | An architectural reference for evaluating VSA + BFF in complex role-based systems |
| Teams | A starting template for building applications with clear separation of concerns |
| Architecture discussions | A concrete example to point at instead of abstract diagrams |

---

## What BonusSystem Actually Does

BonusSystem is a **bonus loyalty platform** — a realistic domain complex enough to stress-test the architecture:

1. **Companies** register on the platform and receive bonus point allocations from system administrators
2. **Companies** register stores that must pass an **admin approval workflow** before activation
3. **Buyers** earn bonus points when making purchases at approved stores
4. **Sellers** (store employees) process transactions by scanning buyer QR codes
5. **Bonus points expire quarterly**, resetting buyer balances (with full expiration audit trail)
6. **Observers** (company-level and system-level) access read-only analytics and reports
7. **System administrators** manage the entire ecosystem — companies, stores, balances, notifications

This isn't a toy CRUD app. The domain includes approval workflows, quarterly expiration mechanisms, role-based permission matrices, and multi-level observation — exactly the kind of complexity where architectural choices matter.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Client (API consumers)                     │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API Layer (Features/)                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Admin/    │  │   Buyers/   │  │  Sellers/   │  ...         │
│  │  Endpoints  │  │  Endpoints  │  │  Endpoints  │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
└─────────┼────────────────┼────────────────┼─────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BFF Layer (Services/BFF/)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   AdminBff   │  │   BuyerBff   │  │  SellerBff   │  ...      │
│  │   Service    │  │   Service    │  │   Service    │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
└─────────┼─────────────────┼─────────────────┼───────────────────┘
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              Repository Layer (Infrastructure/)                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐  │
│  │   User     │  │  Company   │  │   Store    │  │Transaction│  │
│  │ Repository │  │ Repository │  │ Repository │  │Repository │  │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘  └─────┬─────┘  │
└─────────┼───────────────┼───────────────┼──────────────┼────────┘
          └───────────────┴───────────────┴──────────────┘
                                  │
                                  ▼
                    ┌───────────────────────────┐
                    │   PostgreSQL Database     │
                    └───────────────────────────┘
```

### Architectural Principles

1. **Vertical Slices** — features are organised by user role, not by technical layer
2. **BFF Pattern** — each role gets a dedicated Backend for Frontend service that aggregates data and enforces role-specific business logic
3. **Minimal Cross-Slice Dependencies** — slices share only domain entities and repository contracts (interfaces)
4. **Role-Based Permissions** — enforced at both the endpoint level and the BFF service layer

### What Makes the BFF Layer Important

The BFF (Backend for Frontend) layer is the architectural centrepiece. Each role has its own BFF service that:

- **Validates permissions** — checks the user's role before executing any action
- **Aggregates data** — composes responses from multiple repositories without leaking implementation details
- **Encapsulates business logic** — keeps domain rules (e.g., "buyers can only cancel their own transactions") out of endpoint definitions
- **Prevents cross-slice contamination** — the buyer's BFF never calls admin logic, even if both need similar data

---

## Project Structure

```
bonus-system/
├── src/
│   ├── BonusSystem.Api/              # API entry point
│   │   ├── Features/                 # ← VERTICAL SLICES
│   │   │   ├── Admin/                # SystemAdmin role slice
│   │   │   │   ├── AdminEndpoints.cs
│   │   │   │   ├── Requests/         # Admin-specific request models
│   │   │   │   └── Responses/        # Admin-specific response models
│   │   │   ├── Buyers/               # Buyer role slice
│   │   │   │   ├── BuyerEndpoints.cs
│   │   │   │   ├── Requests/
│   │   │   │   └── Responses/
│   │   │   ├── Sellers/              # Seller role slice
│   │   │   ├── Companies/            # Company management slice
│   │   │   ├── Observers/            # Observer role slice
│   │   │   └── Auth/                 # Authentication slice
│   │   ├── Infrastructure/
│   │   │   └── Extensions/           # Middleware, DI registration
│   │   └── Program.cs
│   │
│   ├── BonusSystem.Core/             # Domain logic
│   │   ├── Services/
│   │   │   ├── Interfaces/
│   │   │   └── Implementations/
│   │   │       └── BFF/              # ← ONE BFF SERVICE PER ROLE
│   │   │           ├── BaseBffService.cs
│   │   │           ├── AdminBffService.cs
│   │   │           ├── BuyerBffService.cs
│   │   │           ├── SellerBffService.cs
│   │   │           ├── CompanyBffService.cs
│   │   │           └── ObserverBffService.cs
│   │   └── Repositories/             # ← SHARED CONTRACTS ONLY
│   │       ├── IRepository.cs         # Generic CRUD interface
│   │       ├── IUserRepository.cs
│   │       ├── ICompanyRepository.cs
│   │       ├── IStoreRepository.cs
│   │       └── ITransactionRepository.cs
│   │
│   ├── BonusSystem.Infrastructure/   # Data access implementations
│   │   ├── DataAccess/
│   │   │   ├── EntityFramework/      # EF Core context and entities
│   │   │   └── Repositories/         # Repository implementations
│   │   └── ExternalServices/
│   │
│   └── BonusSystem.Shared/           # Cross-cutting concerns
│       ├── Dtos/                     # Data transfer objects
│       └── Models/                   # Shared domain models
│
├── docker-compose.yml
├── Dockerfile
├── test.http                         # 100+ HTTP requests for manual testing
├── .env                              # Environment configuration
└── README.md
```

### Where to Look

| You want to understand... | Look at... |
|---------------------------|------------|
| How a vertical slice is structured | `Features/Buyers/` — endpoints, requests, responses in one place |
| How BFF services work | `Services/BFF/BuyerBffService.cs` — role-specific logic |
| What slices share (and don't) | `Repositories/` — only interfaces cross slice boundaries |
| How to add a new role/feature | [Adding New Features](#adding-new-features) section below |
| Anti-patterns to avoid | [Anti-Patterns](#anti-patterns-to-avoid) section below |

---

## User Roles & Permissions

The system implements **7 distinct user roles**, each with a dedicated vertical slice and BFF service:

| Role | Enum | Slice | What They Do |
|------|------|-------|-------------|
| **Buyer** | 0 | `Features/Buyers/` | Earn/spend points, view balance, generate QR codes, cancel transactions |
| **Seller** | 1 | `Features/Sellers/` | Scan buyer QR codes, process earn/spend transactions, handle returns |
| **StoreAdmin** | 2 | `Features/Sellers/` | All seller capabilities + store configuration |
| **SystemAdmin** | 3 | `Features/Admin/` | Register companies, approve stores, credit balances, system-wide management |
| **CompanyObserver** | 4 | `Features/Observers/` | Read-only analytics scoped to their company |
| **SystemObserver** | 5 | `Features/Observers/` | Read-only analytics across the entire platform |
| **Company** | 7 | `Features/Companies/` | Register stores, manage sellers, view company statistics |

### Permissions Matrix

| Action | Buyer | Seller | StoreAdmin | Company | CompanyObserver | SystemObserver | SystemAdmin |
|--------|:-----:|:------:|:----------:|:-------:|:---------------:|:--------------:|:-----------:|
| Earn/spend points | ✓ | ✓ | ✓ | — | — | — | — |
| Register stores | — | — | — | ✓ | — | — | ✓ |
| Approve stores | — | — | — | — | — | — | ✓ |
| Credit companies | — | — | — | — | — | — | ✓ |
| View own data | ✓ | ✓ | ✓ | ✓ | — | — | ✓ |
| View company analytics | — | — | — | ✓ | ✓ | — | ✓ |
| View system analytics | — | — | — | — | — | ✓ | ✓ |

---

## Domain Concepts

### Transaction Lifecycle

Transactions flow through typed states that map to real-world bonus programme operations:

**Types:** `Earn` (buyer gets points) · `Spend` (buyer redeems points) · `Expire` (quarterly reset) · `AdminAdjustment` (manual correction)

**Statuses:** `Pending` → `Completed` | `Reversed` | `Failed`

### Store Approval Workflow

```
Company registers store  →  PendingApproval (default)
                                    │
                     Admin calls PUT /stores/{id}/moderate
                                    │
                          ┌─────────┴──────────┐
                          ▼                    ▼
                    Active (approved)    Inactive (rejected)
```

### Company Balance Model

Companies maintain two balance fields — `currentBalance` (real-time available points) and `originalBalance` (baseline for quarterly resets). When an admin credits a company, both fields increment. This dual-balance design ensures quarterly expirations reset buyer balances without losing the company's allocated pool.

### Quarterly Expiration

All buyer bonus points expire quarterly. The system creates `TransactionType.Expire` records for audit trail, zeroes buyer balances, and tracks the full expiration history. Company balances reset to their `originalBalance`.

---

## Quick Start

### Prerequisites

- **Docker** and **Docker Compose** (required)
- **.NET 9 SDK** (optional — only needed for local development without Docker)

### Running with Docker

```bash
# Clone and start
git clone https://github.com/danila-permogorskii/bonus-system.git
cd bonus-system
docker-compose up -d
```

This launches three containers:

| Service | URL | Purpose |
|---------|-----|---------|
| API | [localhost:5001](http://localhost:5001) | .NET 9 Web API (redirects to Swagger) |
| PostgreSQL | localhost:5432 | Database |
| pgAdmin | [localhost:5050](http://localhost:5050) | Database management (email: `admin@bonussystem.com`, password: `admin`) |

> **Note:** This is an API-only project. There is no web frontend. Use Swagger UI or tools like Postman/Insomnia to interact with endpoints.

### Demo Accounts

| Role | Email | Password |
|------|-------|----------|
| Buyer | buyer1@example.com | Password123! |
| Seller | seller1@example.com | Password123! |
| SystemAdmin | admin1@example.com | Password123! |
| CompanyObserver | observer1@example.com | Password123! |

To test Company (role 7) and SystemObserver (role 5) roles, register new users via `POST /auth/register` from an admin account.

### Testing the API

**Swagger UI** (recommended for exploration): Navigate to [localhost:5001/api-docs](http://localhost:5001/api-docs), click "Authorize", and paste your JWT token from the login response.

**test.http** (recommended for workflows): Open in VS Code with the REST Client extension. Contains 100+ pre-configured requests organised by role.

**Postman/Insomnia**: Import the OpenAPI spec from `/api-docs/v1/swagger.json`.

### Stopping

```bash
docker-compose down        # Stop containers
docker-compose down -v     # Stop and remove volumes (fresh start)
```

---

## Development Workflow

### Building Locally

```bash
dotnet restore
dotnet build BonusSystem.sln
cd src/BonusSystem.Api && dotnet run
```

### Database Migrations

```bash
cd src/BonusSystem.Infrastructure
dotnet ef migrations add <MigrationName> --startup-project ../BonusSystem.Api
dotnet ef database update --startup-project ../BonusSystem.Api
```

### Port Configuration

All ports are configured in `.env`:

| Service | Default Port | Variable |
|---------|-------------|----------|
| API | 5001 | `API_HTTP_PORT` |
| PostgreSQL | 5432 | `POSTGRES_PORT` |
| pgAdmin | 5050 | `PGADMIN_PORT` |

---

## Adding New Features

This is where VSA pays off. Adding a new role or feature follows a predictable five-step pattern with no risk of breaking existing slices.

### Example: Adding a "Moderator" Role

**Step 1 — Define the role** in `BonusSystem.Shared/Models/UserRole.cs`:

```csharp
public enum UserRole
{
    // ... existing roles
    Moderator
}
```

**Step 2 — Create the feature slice** at `Features/Moderators/ModeratorEndpoints.cs` with its own `Requests/` and `Responses/` subdirectories.

**Step 3 — Create the BFF service** at `Services/BFF/ModeratorBffService.cs`, inheriting from `BaseBffService`.

**Step 4 — Register in DI** in `ServiceCollectionExtensions.cs`:

```csharp
services.AddScoped<IModeratorBffService, ModeratorBffService>();
```

**Step 5 — Map endpoints** in `Program.cs`:

```csharp
app.MapModeratorEndpoints();
```

Notice the pattern: you never modify existing slices. The new feature is entirely additive.

---

## Anti-Patterns to Avoid

These are the most common ways teams break VSA boundaries. The codebase is designed to demonstrate the correct patterns — study the BFF services to see how.

### Cross-Slice Dependencies

```csharp
// ❌ BAD: Buyer slice importing admin logic
group.MapGet("/admin-data", async (IAdminBffService adminService) => {
    return await adminService.GetSystemStats();
});

// ✅ GOOD: If both roles need stats, each BFF calls repositories independently
```

### Shared Services with Role Logic

```csharp
// ❌ BAD: Single service branching on role
public async Task ProcessAction(Guid userId, string action)
{
    var role = await GetUserRole(userId);
    if (role == UserRole.Admin) { /* ... */ }
    else if (role == UserRole.Buyer) { /* ... */ }
}

// ✅ GOOD: Each role has its own BFF service
// AdminBffService.ProcessAdminAction()
// BuyerBffService.ProcessBuyerAction()
```

### Fat Shared DTOs

```csharp
// ❌ BAD: Universal DTO with fields for every role
public class UniversalUserDto
{
    public decimal BuyerBalance { get; set; }       // Only buyers need this
    public List<Store> SellerStores { get; set; }   // Only sellers need this
    public CompanyStats AdminStats { get; set; }    // Only admins need this
}

// ✅ GOOD: Role-specific response models in each feature folder
// Features/Buyers/Responses/BuyerContextResponse.cs
// Features/Sellers/Responses/SellerContextResponse.cs
```

### Repository Methods with Role Logic

```csharp
// ❌ BAD: Repository aware of roles
Task<IEnumerable<TransactionDto>> GetTransactionsForUser(Guid userId, UserRole role);

// ✅ GOOD: Generic repository methods; BFF decides which to call
Task<IEnumerable<TransactionDto>> GetTransactionsByUserIdAsync(Guid userId);
Task<IEnumerable<TransactionDto>> GetTransactionsByCompanyIdAsync(Guid companyId);
```

---

## Technical Stack

| Layer | Technology |
|-------|-----------|
| Runtime | .NET 9 with Minimal APIs |
| ORM | Entity Framework Core 9 |
| Database | PostgreSQL 17 (Alpine) |
| Authentication | JWT with role-based claims |
| API Documentation | Swagger/OpenAPI |
| Containerisation | Docker & Docker Compose |
| DB Management | pgAdmin 4 |

---

## Further Reading

- [Jimmy Bogard — Vertical Slice Architecture](https://www.jimmybogard.com/vertical-slice-architecture/) — the original articulation of the pattern
- [CQRS and Vertical Slices](https://www.jimmybogard.com/cqrs-and-mediator-patterns/) — how CQRS complements VSA
- [Feature Folders in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/areas) — Microsoft's take on feature-based organisation
- [eShop Reference Application](https://github.com/dotnet/eShop) — Microsoft's microservices reference (different pattern, useful comparison)

---

## Contributing

Contributions are welcome. When adding features, follow the VSA patterns established in the codebase — keep slices self-contained, avoid cross-slice imports, and place business logic in BFF services rather than endpoints.

---

## Licence

MIT — see [LICENCE](LICENCE) for details. Free to use as a reference, fork, or adapt. Attribution appreciated but not required.

---

*Built by [bogdanna.dev](https://bogdanna.dev) as an architectural reference for the .NET community.*