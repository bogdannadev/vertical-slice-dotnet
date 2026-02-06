# BonusSystem: Vertical Slice Architecture Reference Implementation

A **bonus loyalty platform** demonstrating **Vertical Slice Architecture** with Backend for Frontend (BFF) pattern. Designed for multi-role systems where companies run bonus programmes, stores process transactions, and buyers earn/spend points.

**Built with .NET 9 and PostgreSQL** — API-only design with Swagger/OpenAPI documentation.

## What is This Repository?

This project serves as:
- **For Junior Developers**: A practical guide to Vertical Slice Architecture (VSA) with working, production-style code
- **For Senior Developers**: An architectural reference for evaluating VSA + BFF in complex multi-role systems
- **For Teams**: A starting template for building role-based applications with clear separation of concerns

**Key Architectural Decisions:**
- 7 distinct user roles with dedicated BFF services
- Quarterly bonus expiration mechanism
- Store approval workflow (pending → approved/rejected)
- Company balance management with original balance tracking

---

## Table of Contents

- [What BonusSystem Actually Does](#what-bonussystem-actually-does)
- [Architecture Overview](#architecture-overview)
- [Why Vertical Slice Architecture?](#why-vertical-slice-architecture)
- [Project Structure](#project-structure)
- [User Roles & Capabilities](#user-roles--capabilities)
- [Domain Concepts](#domain-concepts)
- [Quick Start](#quick-start)
- [Development Workflow](#development-workflow)
- [Adding New Features](#adding-new-features)
- [Manual Testing](#manual-testing)
- [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
- [Technical Stack](#technical-stack)
- [Contributing](#contributing)
- [Licence](#licence)

---

## What BonusSystem Actually Does

BonusSystem is a **bonus loyalty platform** where:

1. **Companies** register and receive bonus point allocations from system administrators
2. **Companies** register stores that must be **approved by administrators** before activation
3. **Buyers** (end customers) earn bonus points when making purchases at approved stores
4. **Sellers** (store employees) process transactions by scanning buyer QR codes
5. **Bonus points expire quarterly**, resetting buyer balances (with expiration transactions tracked)
6. **Observers** (company/system level) access read-only analytics and reports
7. **Administrators** manage the entire ecosystem: approve stores, credit companies, send notifications

### Real-World Scenario

Imagine a shopping centre consortium:
- The consortium (Admin) registers **Coffee Chain Ltd** with 50,000 bonus points
- Coffee Chain registers three stores: "Downtown Café", "Airport Branch", "Mall Location"
- Admin **approves** these stores (pending → active)
- A buyer makes a £20 purchase at Downtown Café
- The seller scans the buyer's QR code, credits 200 bonus points (10% back)
- At quarter-end, all unused buyer bonuses **expire** (tracked as Expire transactions)
- Company observers analyse: "Downtown Café: 5,000 transactions, 45% redemption rate"

### Transaction Lifecycle

```
Buyer makes purchase
    ↓
Seller scans QR code → TransactionType.Earn (adds points)
    ↓
Transaction saved as TransactionStatus.Completed
    ↓
Buyer's balance updated immediately
    ↓
[90 days later] ProcessExpirationAsync() runs
    ↓
TransactionType.Expire created, balance reset to 0
```

### Key Business Rules Implemented

- **Store Approval Workflow**: New stores start as `StoreStatus.PendingApproval`
- **Company Balance Tracking**: Companies have both `currentBalance` and `originalBalance` (for quarterly resets)
- **Transaction Reversal**: Transactions can be reversed (refunds), adjusting balances accordingly
- **Quarterly Expiration**: All buyer bonuses expire quarterly (implemented in `ProcessExpirationAsync`)
- **Role-Based Permissions**: Each BFF service validates user role before executing actions

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend (React)                        │
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
          │      ┌─────────┴──────────┐     │
          │      │                    │     │
          ▼      ▼                    ▼     ▼
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

1. **Vertical Slices**: Features are organised by user role rather than technical layer
2. **BFF Pattern**: Each role has a dedicated Backend for Frontend service
3. **Minimal Cross-Slice Dependencies**: Slices share only domain entities and repository contracts
4. **Role-Based Permissions**: Enforced at both endpoint and BFF service layers

---

## Why Vertical Slice Architecture?

### The Problem with Traditional Layered Architecture

In traditional N-tier architecture, adding a new feature requires touching multiple layers:

```
❌ Traditional Layers (adds coupling)
   Controller Layer → Service Layer → Repository Layer
   └─ Change admin feature requires modifying UserController,
      UserService, and UserRepository even if buyers don't need it
```

### The VSA Solution

Vertical Slice Architecture organises code by **feature** rather than **technical layer**:

```
✅ Vertical Slices (reduces coupling)
   Admin Feature: AdminEndpoints → AdminBffService → Repositories
   Buyer Feature: BuyerEndpoints → BuyerBffService → Repositories
   └─ Change admin feature only touches Admin slice
```

### Benefits for This Project

| Traditional Layers | Vertical Slice Architecture |
|-------------------|----------------------------|
| Feature changes ripple across layers | Changes stay within one slice |
| Shared services create tight coupling | Minimal cross-slice dependencies |
| Difficult to reason about role permissions | Role logic encapsulated in BFF service |
| Testing requires understanding entire stack | Test individual slices in isolation |

### Trade-offs

**Advantages:**
- Faster feature development (no layer coordination)
- Clearer role-based boundaries
- Easier onboarding (understand one slice at a time)
- Better scalability (split slices into microservices if needed)

**Disadvantages:**
- Some code duplication across slices (by design)
- Requires discipline to avoid cross-slice dependencies
- Shared concerns (auth, logging) need careful abstraction

---

## Project Structure

```
bonus-system/
├── src/
│   ├── BonusSystem.Api/              # API entry point
│   │   ├── Features/                 # ← VERTICAL SLICES
│   │   │   ├── Admin/                # Admin role slice
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
│   │   └── Program.cs                # Application entry point
│   │
│   ├── BonusSystem.Core/             # Domain logic
│   │   ├── Services/
│   │   │   ├── Interfaces/
│   │   │   └── Implementations/
│   │   │       └── BFF/              # ← BFF SERVICES (one per role)
│   │   │           ├── BaseBffService.cs
│   │   │           ├── AdminBffService.cs
│   │   │           ├── BuyerBffService.cs
│   │   │           ├── SellerBffService.cs
│   │   │           ├── CompanyBffService.cs
│   │   │           └── ObserverBffService.cs
│   │   ├── Repositories/             # ← SHARED REPOSITORY CONTRACTS
│   │   │   ├── IUserRepository.cs
│   │   │   ├── ICompanyRepository.cs
│   │   │   ├── IStoreRepository.cs
│   │   │   └── ITransactionRepository.cs
│   │
│   ├── BonusSystem.Infrastructure/   # Data access implementations
│   │   ├── DataAccess/
│   │   │   ├── EntityFramework/      # EF Core context and entities
│   │   │   └── Repositories/         # Repository implementations
│   │   └── ExternalServices/         # Third-party integrations
│   │
│   └── BonusSystem.Shared/           # Cross-cutting concerns
│       ├── Dtos/                     # Data transfer objects
│       └── Models/                   # Shared domain models
│
├── docker-compose.yml                # Multi-container orchestration
├── Dockerfile                        # API container build
├── test.http                         # HTTP request collection for manual testing
├── .env                              # Environment variables
└── README.md
```

### Key Directories Explained

- **`Features/`**: Each subdirectory is a complete vertical slice containing endpoints, request/response models
- **`Services/BFF/`**: Backend for Frontend services that aggregate data and enforce role-specific business logic
- **`Repositories/`**: Shared data access contracts (interfaces only, implementations in Infrastructure)
- **`Shared/`**: DTOs and models used across multiple slices (kept minimal to avoid coupling)
- **`test.http`**: Manual HTTP test requests for all endpoints (use with VS Code REST Client extension)

---

## User Roles & Capabilities

The system implements **7 distinct user roles**, each with dedicated endpoints and BFF services:

### 1. Buyer (UserRole.Buyer = 0)
**Who**: End customers earning and spending bonus points  
**Slice**: `Features/Buyers/`, `BuyerBffService`  
**Capabilities**:
- View bonus balance with earned/spent/expiring breakdown
- View complete transaction history
- Generate QR code for in-store scanning
- Cancel own transactions (reverses balance)
- Find stores by category

**Key Endpoints**:
- `GET /api/buyers/context` - Get user context + permitted actions
- `GET /api/buyers/balance` - Get bonus summary (total earned, spent, expiring next quarter)
- `GET /api/buyers/transactions` - Full transaction history
- `GET /api/buyers/qrcode` - Generate QR code string (`BONUS-USER-{userId}`)
- `POST /api/buyers/transactions/{id}/cancel` - Reverse a transaction
- `GET /api/buyers/stores?category={category}` - Find participating stores

### 2. Seller (UserRole.Seller = 1)
**Who**: Store employees processing transactions  
**Slice**: `Features/Sellers/`, `SellerBffService`  
**Capabilities**:
- Scan buyer QR codes (receives userId)
- Process transactions: Earn (add points) or Spend (redeem points)
- View store-specific transaction history
- Confirm transaction returns/refunds

**Key Endpoints**:
- `GET /api/sellers/context`
- `POST /api/sellers/transactions/process` - Process buyer transaction
    - Request: `{ buyerId, bonusAmount, totalCost, type: Earn/Spend }`
    - Validates buyer balance for Spend transactions
- `POST /api/sellers/transactions/{id}/return` - Confirm return/refund
- `GET /api/sellers/store/transactions` - Store transaction history

### 3. StoreAdmin (UserRole.StoreAdmin = 2)
**Who**: Store managers with elevated store permissions  
**Slice**: Same as Seller, extended permissions  
**Capabilities**: All Seller capabilities + store configuration (implementation placeholder)

### 4. SystemAdmin (UserRole.SystemAdmin = 3)
**Who**: Platform administrators managing the entire ecosystem  
**Slice**: `Features/Admin/`, `AdminBffService`  
**Capabilities**:
- Register companies (with or without admin user)
- Update company status: Active/Suspended/Pending
- **Approve/reject store registrations** (moderation workflow)
- Credit company bonus pools
- Get system-wide transactions (filterable by company, date range)
- Send notifications (to specific users or by role)
- Calculate transaction fees per company

**Key Endpoints**:
- `POST /api/admin/companies` - Register company (legacy)
- `POST /api/admin/companies-with-admin` - Register company + create admin user
- `PUT /api/admin/companies/{id}/status` - Update company status (0=Active, 1=Suspended, 2=Pending)
- `PUT /api/admin/stores/{id}/moderate?approve={true|false}` - **Approve/reject store**
- `POST /api/admin/companies/{id}/credit` - Credit company balance (adds to both current and original balance)
- `GET /api/admin/transactions?companyId=&startDate=&endDate=` - System transactions
- `POST /api/admin/notifications?recipientId=&message=&type=` - Send notifications
- `GET /api/admin/transaction-fees?fromDate=&endDate=&feePercent=` - Calculate fees

### 5. Company (UserRole.Company = 7)
**Who**: Company representatives managing their stores and sellers  
**Slice**: `Features/Companies/`, `CompanyBffService`  
**Capabilities**:
- Register new stores (go to PendingApproval status)
- Register sellers and assign to stores
- View company statistics (transaction volume, active stores)
- View all company stores with assigned sellers (paginated, filterable)

**Key Endpoints**:
- `POST /api/companies/stores` - Register new store (requires admin approval)
- `POST /api/companies/sellers` - Register seller, optionally assign to store
- `GET /api/companies/statistics` - Company-level stats
- `GET /api/companies/stores-with-sellers?page=&pageSize=&storeStatus=` - Paginated store list

### 6. CompanyObserver (UserRole.CompanyObserver = 4)
**Who**: Company-level read-only analysts  
**Slice**: `Features/Observers/`, `ObserverBffService`  
**Capabilities**:
- View company-specific statistics
- View transaction summaries for their company
- Access read-only reports

**Key Endpoints**:
- `GET /api/observers/statistics?companyId={id}&startDate=&endDate=` - Filter by company
- `GET /api/observers/transactions/summary?companyId={id}` - Transaction breakdown by type/status
- `GET /api/observers/companies` - Companies overview (if SystemObserver)

### 7. SystemObserver (UserRole.SystemObserver = 5)
**Who**: Platform-level read-only analysts  
**Slice**: Same as CompanyObserver, no company filter required  
**Capabilities**:
- View system-wide statistics (all companies)
- View transaction summaries across entire platform
- Access companies overview with aggregated metrics

**Permissions Matrix**:

| Action | Buyer | Seller | StoreAdmin | Company | CompanyObserver | SystemObserver | SystemAdmin |
|--------|-------|--------|------------|---------|-----------------|----------------|-------------|
| Earn/spend points | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Register stores | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| Approve stores | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Credit companies | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| View own data | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| View company analytics | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ✅ |
| View system analytics | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |

---

## Domain Concepts

### Transaction Types (TransactionType enum)

```csharp
public enum TransactionType
{
    Earn = 0,           // Buyer earns points during purchase
    Spend = 1,          // Buyer redeems points
    Expire = 2,         // Quarterly expiration (automated)
    AdminAdjustment = 3 // Manual admin correction
}
```

### Transaction Statuses (TransactionStatus enum)

```csharp
public enum TransactionStatus
{
    Pending = 0,    // Transaction initiated, not yet completed
    Completed = 1,  // Transaction successful
    Reversed = 2,   // Transaction cancelled/refunded
    Failed = 3      // Transaction failed validation
}
```

### Store Approval Workflow

```
Company registers store
    ↓
StoreStatus.PendingApproval (default)
    ↓
Admin calls PUT /api/admin/stores/{id}/moderate?approve=true
    ↓
StoreStatus.Active (approved) or StoreStatus.Inactive (rejected)
```

### Company Balance Model

Companies have two balance fields:
- **currentBalance**: Real-time available bonus points
- **originalBalance**: Baseline for quarterly resets

When admin credits a company:
```csharp
currentBalance += amount;
originalBalance += amount;  // Persists through expirations
```

### Quarterly Expiration Mechanism

```csharp
// Runs quarterly (implementation in EntityFrameworkTransactionRepository)
await ProcessExpirationAsync(DateTime.UtcNow);

// For each buyer with balance > 0:
// 1. Create TransactionType.Expire transaction
// 2. Set balance to 0
// 3. Record expiration in transaction history
```

### Notification System

4 notification types:
- **Transaction**: Sent after earn/spend events
- **System**: Platform-wide announcements
- **Expiration**: Warnings before quarterly expiration
- **AdminMessage**: Direct messages from admins

---

## Quick Start

### Prerequisites

- **Docker** and **Docker Compose** (required)
- **.NET 9 SDK** (optional, for local development without Docker)

### Running with Docker

1. **Clone the repository:**
   ```bash
   git clone https://github.com/danila-permogorskii/bonus-system.git
   cd bonus-system
   ```

2. **Start all services:**
   ```bash
   docker-compose up -d
   ```

   This launches:
    - **API**: .NET 9 Web API ([http://localhost:5001](http://localhost:5001))
    - **PostgreSQL**: Database (localhost:5432)
    - **pgAdmin**: Database management UI ([http://localhost:5050](http://localhost:5050))

3. **Access the services:**
    - **API Swagger Documentation**: [http://localhost:5001/api-docs](http://localhost:5001/api-docs)
    - **API Root**: [http://localhost:5001](http://localhost:5001) (redirects to /api-docs)
    - **pgAdmin**: [http://localhost:5050](http://localhost:5050)
        - Email: `admin@bonussystem.com`
        - Password: `admin`

**Note**: This is an **API-only** project. There is no web frontend. Use Swagger UI or tools like Postman/Insomnia to interact with endpoints.

### Demo Accounts

Test accounts for each major role:

| Role | UserRole Enum | Email | Password |
|------|---------------|-------|----------|
| Buyer | 0 | buyer1@example.com | Password123! |
| Seller | 1 | seller1@example.com | Password123! |
| SystemAdmin | 3 | admin1@example.com | Password123! |
| CompanyObserver | 4 | observer1@example.com | Password123! |

**Note**: To test Company (role 7) and SystemObserver (role 5) roles, register new users via:
- `POST /auth/register` (requires authentication from admin account)
- Specify `role: 7` for Company or `role: 5` for SystemObserver in request body

### Testing the API

Since this is an API-only project, you can test endpoints using:

#### 1. Swagger UI (Recommended for Exploration)
Navigate to [http://localhost:5001/api-docs](http://localhost:5001/api-docs) after starting docker-compose.
- Click "Authorize" button, paste JWT token from login response
- Expand endpoint groups (Buyers, Sellers, Admin, etc.)
- Use "Try it out" to execute requests directly

#### 2. test.http File (Recommended for Workflows)
Open `test.http` in VS Code with REST Client extension:
- Contains pre-configured requests for all endpoints
- Organised by role: Authentication, Buyers, Sellers, Admin, Company, Observers
- Includes example payloads and token management
- Execute requests with Ctrl+Alt+R (or Cmd+Alt+R on Mac)

**Example workflow:**
```http
# 1. Login as buyer
POST {{baseUrl}}/auth/login
Content-Type: application/json

{
    "email": "buyer1@example.com",
    "password": "Password123!"
}

# 2. Copy token from response, set buyerToken variable

# 3. Get buyer balance
GET {{baseUrl}}/api/buyers/balance
Authorization: Bearer {{buyerToken}}
```

#### 3. Postman/Insomnia
Import the OpenAPI spec from `/api-docs/v1/swagger.json` or manually configure requests.

### Stopping the Environment

```bash
# Stop containers
docker-compose down

# Stop and remove volumes (fresh start)
docker-compose down -v
```

---

## Development Workflow

### Building Locally

```bash
# Restore dependencies
dotnet restore

# Build solution
dotnet build BonusSystem.sln

# Run API locally (requires PostgreSQL connection)
cd src/BonusSystem.Api
dotnet run
```

### Database Migrations

```bash
# Add a new migration
cd src/BonusSystem.Infrastructure
dotnet ef migrations add <MigrationName> --startup-project ../BonusSystem.Api

# Apply migrations
dotnet ef database update --startup-project ../BonusSystem.Api
```

### Port Configuration

All ports are configured in the `.env` file:

| Service | Port | Configuration Variable |
|---------|------|------------------------|
| API | 5001 | `API_HTTP_PORT=5001` |
| PostgreSQL | 5432 | (default PostgreSQL port) |
| pgAdmin | 5050 | `PGADMIN_PORT=5050` |

---

## Adding New Features

This is where VSA shines. To add a new feature or role, follow these steps:

### Example: Adding a "Moderator" Role

#### Step 1: Define the Role

Add the role to `BonusSystem.Shared/Models/UserRole.cs`:

```csharp
public enum UserRole
{
    // ... existing roles
    Moderator
}
```

#### Step 2: Create the Feature Slice

Create `src/BonusSystem.Api/Features/Moderators/ModeratorEndpoints.cs`:

```csharp
public static class ModeratorEndpoints
{
    public static void MapModeratorEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/moderators")
            .RequireAuthorization()
            .WithTags("Moderators");

        group.MapGet("/context", GetModeratorContext);
        group.MapGet("/pending-approvals", GetPendingApprovals);
        // ... more endpoints
    }

    private static async Task<IResult> GetModeratorContext(
        Guid userId,
        IModeratorBffService moderatorService)
    {
        var context = await moderatorService.GetUserContextAsync(userId);
        return Results.Ok(context);
    }
}
```

#### Step 3: Create the BFF Service

Create `src/BonusSystem.Core/Services/Implementations/BFF/ModeratorBffService.cs`:

```csharp
public class ModeratorBffService : BaseBffService, IModeratorBffService
{
    public ModeratorBffService(
        IDataService dataService,
        IAuthenticationService authService)
        : base(dataService, authService)
    {
    }

    public override async Task<IEnumerable<PermittedActionDto>> GetPermittedActionsAsync(Guid userId)
    {
        var role = await _dataService.Users.GetUserRoleAsync(userId);
        if (role != UserRole.Moderator)
            return Enumerable.Empty<PermittedActionDto>();

        return new List<PermittedActionDto>
        {
            new() { ActionName = "ApproveStore", Description = "Approve store requests" },
            new() { ActionName = "RejectStore", Description = "Reject store requests" }
        };
    }

    public async Task<IEnumerable<StoreDto>> GetPendingApprovalsAsync()
    {
        // Business logic for moderator-specific operations
    }
}
```

#### Step 4: Register in DI Container

Add to `src/BonusSystem.Api/Infrastructure/Extensions/ServiceCollectionExtensions.cs`:

```csharp
services.AddScoped<IModeratorBffService, ModeratorBffService>();
```

#### Step 5: Map Endpoints

Add to `src/BonusSystem.Api/Program.cs`:

```csharp
app.MapModeratorEndpoints();
```

#### Step 6: Test Manually

Test the new slice using `test.http`:

```http
### Moderator Login
POST {{baseUrl}}/auth/login
Content-Type: application/json

{
    "email": "moderator@example.com",
    "password": "Password123!"
}

### Get Pending Approvals
GET {{baseUrl}}/api/moderators/pending-approvals
Authorization: Bearer {{moderatorToken}}
```

Or use Swagger UI at `/api-docs`.

### Key Principles When Adding Features

1. **Self-Contained Slices**: Keep all feature-specific code within the slice directory
2. **No Cross-Slice References**: Avoid importing from other feature slices
3. **Share Via Repositories**: Use repository interfaces for data access, never direct EF context calls
4. **BFF for Business Logic**: Complex logic goes in BFF service, not endpoints
5. **Minimal Shared Models**: Only share truly common DTOs (avoid premature abstraction)

---

## Manual Testing

**This project has no automated test suite.** All testing is done manually via:

### 1. Swagger UI (Browser-Based)
Navigate to [http://localhost:5001/api-docs](http://localhost:5001/api-docs):
- Interactive API documentation
- "Try it out" buttons for immediate testing
- Built-in authentication (click "Authorize", paste JWT token)
- Organised by feature slice (Buyers, Sellers, Admin, etc.)

### 2. test.http File (VS Code REST Client)
Contains 100+ pre-configured HTTP requests in `test.http`:
```http
# Example: Login and get buyer balance
POST {{baseUrl}}/auth/login
Content-Type: application/json

{
    "email": "buyer1@example.com",
    "password": "Password123!"
}

### Copy token from response

GET {{baseUrl}}/api/buyers/balance
Authorization: Bearer {{buyerToken}}
```

**Usage**: Install "REST Client" extension in VS Code, then click "Send Request" above any `###` separator.

### 3. External Tools
- **Postman**: Import OpenAPI spec from `/api-docs/v1/swagger.json`
- **Insomnia**: Same OpenAPI import
- **curl**: Manual command-line requests

### Future: Adding Automated Tests

To add test coverage to this project, you would:
1. Create `tests/` folder with xUnit projects
2. Add `BonusSystem.Api.Tests`, `BonusSystem.Core.Tests`, `BonusSystem.Infrastructure.Tests`
3. Use Moq for mocking repositories and services
4. Consider Testcontainers for integration tests with real PostgreSQL

**Recommended structure** (not currently implemented):
```
tests/
├── BonusSystem.Api.Tests/       # Endpoint integration tests
├── BonusSystem.Core.Tests/      # BFF service unit tests
└── BonusSystem.Infrastructure.Tests/  # Repository integration tests
```

---

## Anti-Patterns to Avoid

### ❌ Cross-Slice Dependencies

**Bad:**
```csharp
// In BuyerEndpoints.cs
public static void MapBuyerEndpoints(this IEndpointRouteBuilder app)
{
    // DON'T import AdminBffService in Buyer slice
    group.MapGet("/admin-data", async (IAdminBffService adminService) => {
        return await adminService.GetSystemStats();
    });
}
```

**Good:**
```csharp
// If both roles need system stats, create a shared StatsService
// or have each BFF service call the repository independently
```

### ❌ Shared Services with Role Logic

**Bad:**
```csharp
// UserService.cs (single service for all roles)
public async Task ProcessAction(Guid userId, string action)
{
    var role = await GetUserRole(userId);
    if (role == UserRole.Admin) { /* admin logic */ }
    else if (role == UserRole.Buyer) { /* buyer logic */ }
    // ← This creates coupling and violates single responsibility
}
```

**Good:**
```csharp
// AdminBffService.cs
public async Task ProcessAdminAction(Guid userId) { /* admin logic */ }

// BuyerBffService.cs  
public async Task ProcessBuyerAction(Guid userId) { /* buyer logic */ }
```

### ❌ Fat DTOs Across Slices

**Bad:**
```csharp
// Shared/Dtos/UniversalUserDto.cs
public class UniversalUserDto
{
    // Properties needed by ALL roles (creates bloat)
    public decimal BuyerBalance { get; set; }          // Only buyers need this
    public List<Store> SellerStores { get; set; }     // Only sellers need this
    public CompanyStats AdminStats { get; set; }       // Only admins need this
}
```

**Good:**
```csharp
// Create role-specific response models in feature folders
// Features/Buyers/Responses/BuyerContextResponse.cs
// Features/Sellers/Responses/SellerContextResponse.cs
```

### ❌ Repository Methods with Role Logic

**Bad:**
```csharp
// ITransactionRepository.cs
Task<IEnumerable<TransactionDto>> GetTransactionsForUser(Guid userId, UserRole role);
// ← Repository shouldn't know about role logic
```

**Good:**
```csharp
// ITransactionRepository.cs (generic)
Task<IEnumerable<TransactionDto>> GetTransactionsByUserIdAsync(Guid userId);
Task<IEnumerable<TransactionDto>> GetTransactionsByCompanyIdAsync(Guid companyId);

// BFF service decides which method to call based on role
```

---

## Technical Stack

### Backend & API
- **.NET 9**: Web API with minimal APIs and endpoint routing
- **Entity Framework Core 9**: ORM for PostgreSQL with migrations
- **PostgreSQL 17**: Primary database (Alpine container)
- **JWT Authentication**: Stateless auth tokens with role-based claims
- **Swagger/OpenAPI**: Interactive API documentation and testing

### Infrastructure
- **Docker & Docker Compose**: Multi-container development environment
- **pgAdmin 4**: Web-based PostgreSQL administration
- **test.http**: HTTP request collection for endpoint testing (VS Code REST Client)

### Development Tools
- **Swagger UI**: Browser-based API exploration at `/api-docs`
- **JWT Tokens**: Manual authentication testing via Authorization headers
- **test.http**: VS Code REST Client file with 100+ pre-configured endpoint requests

**Note**: This project currently has **no automated tests**. Testing is done manually via Swagger UI or the `test.http` file.

---

## Licence

This project is licenced under the **MIT Licence** – see the [LICENCE](LICENCE) file for details.

**TL;DR**: You're free to use this code as a reference, fork it, or adapt it for your own projects. Attribution is appreciated but not required.

---

**Built with ❤️ as an educational resource.**