# Go Coding Style & Architecture Skill

## Purpose

This skill defines how an AI coding agent should design, write, modify, and review Go code.

The objective is to make AI-generated Go code feel like it was written by the same engineering team across:

* fresh Go repositories
* existing Go repositories
* modular monoliths
* REST APIs
* background services
* third-party integrations
* PostgreSQL-backed applications
* domain-oriented modules

The AI must prioritize:

1. Correctness
2. Simplicity
3. Explicit dependencies
4. Clear package boundaries
5. Idiomatic Go
6. Maintainability
7. Production-grade architecture without unnecessary overengineering

The AI must adapt to an existing repository instead of blindly applying this skill.

---

# 1. Core Philosophy

Write Go code that is:

```text
Simple
Explicit
Composable
Dependency-aware
Easy to reason about
Easy to modify
```

Avoid code that is:

```text
Over-abstracted
Framework-driven
Magic-heavy
Generic for no reason
Prematurely optimized
Enterprise-pattern-heavy
```

Prefer:

```go
small interfaces
small structs
explicit dependencies
explicit error handling
constructor injection
package-level ownership
```

over:

```text
global state
service locators
large interfaces
deep inheritance-like abstractions
generic repositories
generic service frameworks
```

---

# 2. Existing Repository Takes Priority

When working in an existing Go repository:

## First inspect

Before writing code, inspect:

```text
go.mod
main.go / cmd/
internal/
pkg/
application wiring
database layer
configuration
existing modules
existing naming conventions
existing error handling
existing repository patterns
existing HTTP patterns
```

Understand the existing architecture before modifying it.

### Rule

Do not introduce a new architectural style into an existing repository if an established style already exists.

For example, if the repository already uses:

```go
type Executor interface {
    Exec(...)
    Query(...)
    QueryRow(...)
}
```

and repositories depend on `Executor`, continue using that abstraction.

Do not introduce a second database abstraction merely because it looks cleaner.

---

# 3. Fresh Repository Initialization

For a new repository, establish a simple structure.

Preferred baseline:

```text
project/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── config/
│   ├── database/
│   └── <domain>/
├── pkg/
├── migrations/
├── go.mod
└── README.md
```

Do not create every possible directory up front.

Create directories when the responsibility actually exists.

Avoid:

```text
internal/
├── controllers/
├── services/
├── repositories/
├── models/
├── utils/
├── helpers/
├── managers/
├── factories/
└── common/
```

unless the repository genuinely requires those boundaries.

Prefer domain-oriented packages:

```text
internal/
└── catalog/
    ├── product/
    ├── category/
    ├── brand/
    └── sync/
```

over technical-layer dumping grounds.

---

# 4. Package Design

Packages should represent meaningful responsibilities.

Good:

```text
internal/catalog/product
internal/catalog/category
internal/catalog/brand
internal/catalog/sync
```

Avoid:

```text
internal/catalog/utils
internal/catalog/helpers
internal/common
internal/shared
```

unless the abstraction genuinely has multiple consumers and a clear ownership boundary.

A package should have a clear answer to:

> What responsibility does this package own?

If the answer is unclear, the package is probably too broad.

---

# 5. Domain-Oriented Architecture

For business-heavy applications, organize around domains rather than technical layers.

Preferred:

```text
internal/
└── catalog/
    ├── product/
    ├── brand/
    ├── category/
    ├── tag/
    ├── attribute/
    └── sync/
```

Not:

```text
internal/
├── handlers/
├── services/
├── repositories/
├── models/
└── mappers/
```

The second structure spreads one business capability across many unrelated directories.

---

# 6. Layer Responsibilities

When a module needs multiple layers, use explicit responsibility boundaries.

Typical flow:

```text
HTTP Handler
     ↓
Application / Service
     ↓
Repository
     ↓
Database
```

For external integrations:

```text
HTTP Handler
     ↓
Application Service
     ↓
Domain / Mapper
     ↓
Integration Client
     ↓
External API
```

Do not allow:

```text
Handler → Database
Handler → External API
Repository → HTTP API
Mapper → Database
Domain → HTTP framework
```

unless there is a deliberate reason.

---

# 7. Interfaces

Define interfaces where they are consumed.

Prefer:

```go
type Repository interface {
    Find(ctx context.Context, id uuid.UUID) (*Product, error)
    Create(ctx context.Context, product *Product) error
    Update(ctx context.Context, product *Product) error
}
```

Avoid large interfaces:

```go
type Repository interface {
    Create(...)
    Update(...)
    Delete(...)
    Find(...)
    FindAll(...)
    Search(...)
    Count(...)
    Export(...)
    Import(...)
    Sync(...)
    Archive(...)
    Restore(...)
}
```

If responsibilities are different, split the interface.

---

# 8. Do Not Create Interfaces Automatically

Do not create an interface simply because:

> Go best practice says everything should have an interface.

Use an interface when:

* dependency substitution matters
* a package boundary requires abstraction
* multiple implementations are expected
* the consumer only needs a subset of behavior
* the abstraction improves dependency direction

Concrete types are acceptable.

Example:

```go
type PostgresRepository struct {
    db database.Executor
}
```

is perfectly valid.

---

# 9. Constructor Pattern

Use explicit constructors.

Preferred:

```go
func NewPostgresRepository(db database.Executor) *PostgresRepository {
    return &PostgresRepository{
        db: db,
    }
}
```

For services:

```go
func NewService(
    productRepo product.Repository,
    categoryRepo category.Repository,
) *Service {
    return &Service{
        productRepo:  productRepo,
        categoryRepo: categoryRepo,
    }
}
```

Avoid constructors that perform business operations.

Constructor responsibility:

```text
create object
validate required configuration if appropriate
wire dependencies
```

Not:

```text
call database
call external API
create business records
```

---

# 10. Dependency Injection

Dependencies must be explicit.

Preferred:

```go
type Service struct {
    repository Repository
    client     ExternalClient
}
```

Construct through:

```go
func NewService(
    repository Repository,
    client ExternalClient,
) *Service
```

Avoid global dependencies:

```go
var db *pgxpool.Pool
var client *http.Client
```

Avoid service locators:

```go
container.Resolve("repository")
```

unless the existing application already has a strong reason for such a mechanism.

---

# 11. Context

For I/O operations, accept `context.Context`.

Preferred:

```go
func (r *PostgresRepository) Find(
    ctx context.Context,
    id uuid.UUID,
) (*Product, error)
```

Use:

```go
ctx
```

for:

* PostgreSQL queries
* HTTP requests
* external APIs
* long-running application operations

Do not store request context inside structs.

Avoid:

```go
type Service struct {
    ctx context.Context
}
```

---

# 12. Error Handling

Errors are values.

Handle them explicitly.

Preferred:

```go
product, err := r.Find(ctx, id)
if err != nil {
    return nil, fmt.Errorf("find product %s: %w", id, err)
}
```

Use `%w` when the underlying error should remain discoverable.

Use:

```go
errors.Is(err, target)
errors.As(err, &target)
```

when callers need to inspect the underlying error.

---

# 13. Error Context

Errors should describe the operation that failed.

Bad:

```go
return err
```

Better:

```go
return fmt.Errorf("create product: %w", err)
```

Best when useful:

```go
return fmt.Errorf(
    "create product %s: %w",
    product.ID,
    err,
)
```

Do not create extremely long error chains that repeat the same information at every layer.

Each layer should add meaningful context.

---

# 14. Sentinel Errors

Use sentinel errors for conditions that callers need to identify.

Example:

```go
var ErrNotFound = errors.New("not found")
```

Then:

```go
if errors.Is(err, ErrNotFound) {
    ...
}
```

Do not create sentinel errors for every possible failure.

Prefer typed errors when structured information is needed.

---

# 15. Domain Errors

Business-level errors should live close to the domain that owns them.

Example:

```go
var ErrInvalidProduct = errors.New("invalid product")
```

Avoid putting all application errors into:

```text
internal/errors
```

unless the error truly belongs to the entire application.

---

# 16. HTTP Error Translation

Do not leak database or infrastructure errors directly to API consumers.

Example:

```text
Repository error
      ↓
Service error
      ↓
HTTP handler
      ↓
HTTP response
```

For example:

```text
ErrNotFound → 404
validation error → 400
unauthorized → 401
forbidden → 403
conflict → 409
unexpected error → 500
```

The exact mechanism should follow the existing application's error-handling architecture.

---

# 17. Struct Design

Keep structs focused.

Prefer:

```go
type Product struct {
    ID       uuid.UUID
    Name     string
    Slug     string
    Status   string
}
```

Avoid giant structs containing unrelated concerns.

Do not automatically expose database structs as API responses.

---

# 18. Domain Models vs DTOs

Separate representations when their responsibilities differ.

Example:

```text
External API
      ↓
WooProduct
      ↓
MappedProduct
      ↓
Domain/Product
      ↓
Database
```

For HTTP:

```text
HTTP Request
      ↓
Request DTO
      ↓
Application Service
      ↓
Domain
```

Response:

```text
Domain / Query Result
      ↓
Response DTO
      ↓
JSON
```

Do not reuse one struct for:

```text
database
external API
domain
HTTP request
HTTP response
```

just to reduce the number of files.

---

# 19. External API Models

External API models should represent the external system.

Example:

```go
type WooProduct struct {
    ID       int64  `json:"id"`
    Name     string `json:"name"`
    Slug     string `json:"slug"`
    Price    string `json:"price"`
}
```

Do not pollute the domain model with WooCommerce-specific fields.

Avoid:

```go
type Product struct {
    WooCommerceID int64
    WooCommerceURL string
}
```

unless those fields genuinely belong to the internal domain.

Use external mapping tables or dedicated mapping structures when external identity must be persisted.

---

# 20. External ID Rule

External IDs must remain distinct from internal IDs.

Example:

```text
WooCommerce:
product ID = 12345

Gluzo:
product ID = UUID
```

Persist the relationship:

```text
product_external_mappings
```

Do not make:

```go
Product.ID = WooProduct.ID
```

unless the external system is explicitly the source of identity for the internal domain.

---

# 21. Mapping

Mapping should be deterministic and side-effect free.

Good:

```go
func MapProduct(source WooProduct) MappedProduct
```

The mapper should not:

```text
query database
call APIs
start transactions
create records
```

Mapping responsibility:

```text
External representation
        ↓
Internal representation
```

---

# 22. Repository Design

Repositories own persistence operations.

Example:

```go
type Repository interface {
    Find(ctx context.Context, id uuid.UUID) (*Product, error)
    Create(ctx context.Context, product *Product) error
    Update(ctx context.Context, product *Product) error
}
```

Repositories should not contain business orchestration.

Avoid:

```go
func (r *Repository) SyncWooCommerceProduct(...)
```

if synchronization involves multiple repositories and business steps.

That belongs in a service.

---

# 23. Database Executor Abstraction

When PostgreSQL transactions are required, prefer an executor abstraction if it is already established.

Example:

```go
type Executor interface {
    Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
    Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
    QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}
```

Then:

```go
type PostgresRepository struct {
    db Executor
}
```

Both:

```text
*pgxpool.Pool
pgx.Tx
```

can satisfy the interface.

This allows repositories to participate in transactions without creating separate repository implementations.

Do not introduce this abstraction if the existing repository already has a different, coherent transaction design.

---

# 24. Transaction Boundaries

Transactions belong to business operations, not individual repository methods.

Good:

```text
Service
  ↓
BEGIN
  ↓
Repository A
Repository B
Repository C
  ↓
COMMIT
```

Avoid:

```text
Repository A → BEGIN/COMMIT
Repository B → BEGIN/COMMIT
Repository C → BEGIN/COMMIT
```

when all operations must succeed or fail together.

---

# 25. Transaction-Scoped Repositories

When using `pgx.Tx`, repositories should use the same transaction:

```go
tx, err := db.Begin(ctx)
if err != nil {
    return err
}

repo := product.NewPostgresRepository(tx)
```

Every repository involved in that business transaction must use the same transaction executor.

Do not accidentally mix:

```text
Product Repository → tx
Category Repository → db pool
Tag Repository → tx
```

because this breaks atomicity.

---

# 26. SQL

Keep SQL explicit and readable.

Prefer:

```sql
SELECT
    id,
    name,
    slug
FROM products
WHERE id = $1
```

over excessively compressed SQL.

Use parameterized queries.

Never construct SQL using raw user input.

Bad:

```go
query := "SELECT * FROM products WHERE name = '" + name + "'"
```

Good:

```go
query := `
    SELECT id, name, slug
    FROM products
    WHERE name = $1
`
```

---

# 27. Dynamic SQL

Dynamic SQL requires special care.

User-controlled values must never directly become:

```text
column names
table names
ORDER BY expressions
SQL fragments
```

For sorting, use a whitelist:

```go
var allowedSorts = map[string]string{
    "name":       "name",
    "created_at": "created_at",
    "price":      "price",
}
```

Then construct SQL only from trusted values.

---

# 28. SQL Scanning

Keep scan order aligned with SELECT order.

Example:

```go
err := row.Scan(
    &product.ID,
    &product.Name,
    &product.Slug,
)
```

Do not hide simple database scans behind excessive generic abstractions.

---

# 29. PostgreSQL Types

Use PostgreSQL types intentionally.

Examples:

```text
UUID        → uuid.UUID
TIMESTAMPTZ → time.Time
NUMERIC     → appropriate decimal representation
JSONB       → explicit JSON type/structure
```

Do not use strings everywhere simply because PostgreSQL can return text.

---

# 30. Money

Never use `float64` for financial values.

Preferred domain representation:

```go
type Money struct {
    Amount   int64
    Currency string
}
```

For INR:

```text
₹699.99
```

represent as:

```text
Amount = 69999
```

PostgreSQL may use:

```sql
NUMERIC(12,2)
```

Conversion should be explicit.

Avoid:

```go
float64
```

for prices, totals, discounts, or financial calculations.

---

# 31. UUIDs

Use UUIDs for internal entity identifiers when that is the established domain design.

Example:

```go
ID uuid.UUID
```

Do not convert UUIDs to strings unnecessarily.

Use strings at external boundaries only when required.

---

# 32. Time

Use:

```go
time.Time
```

for timestamps.

Use pointers where nullability is meaningful:

```go
UpdatedAt *time.Time
```

Do not use:

```go
string
```

for internal timestamps unless interacting with an external API that requires strings.

---

# 33. JSON

Use JSON tags at serialization boundaries.

Example:

```go
type ProductResponse struct {
    ID   uuid.UUID `json:"id"`
    Name string    `json:"name"`
}
```

Do not add JSON tags to every internal struct automatically.

---

# 34. HTTP Clients

External clients should encapsulate:

* base URL
* credentials
* HTTP client
* timeout
* request construction
* response decoding
* external API errors

Example:

```go
type HTTPWooCommerceClient struct {
    baseURL        string
    consumerKey    string
    consumerSecret string
    httpClient     *http.Client
}
```

Use dependency injection for `http.Client`.

---

# 35. HTTP Client Timeouts

Never create external HTTP clients with no timeout unless there is a deliberate reason.

Prefer:

```go
http.Client{
    Timeout: timeout,
}
```

Respect context cancellation as well.

---

# 36. HTTP Status Handling

Do not decode a successful response blindly before checking status.

Expected flow:

```text
HTTP request
    ↓
Check status
    ↓
2xx → decode success body
non-2xx → decode/construct API error
```

External API errors should preserve useful status/body information where appropriate.

---

# 37. Pagination

When consuming paginated external APIs:

* understand how pagination works
* preserve source IDs
* avoid duplicates
* avoid skipped records
* stop deterministically

If response headers contain pagination metadata and the caller needs it, the HTTP client should expose that metadata explicitly.

Do not hide required response metadata just to keep a helper function small.

---

# 38. Service Layer

A service coordinates business operations.

Example:

```go
type Service struct {
    productRepo  product.Repository
    categoryRepo category.Repository
    wooClient    WooCommerceClient
}
```

A service may:

```text
validate
coordinate
map
start transactions
call repositories
call integrations
apply business rules
```

A service should not become a dumping ground for every function in the application.

---

# 39. Avoid Giant Services

If a service grows into:

```text
SyncProduct
SyncCategory
SyncBrand
CreateOrder
CancelOrder
UpdateInventory
SendEmail
GenerateInvoice
```

split responsibilities according to domain boundaries.

---

# 40. Query / Read Side

For read-heavy APIs, do not automatically reuse write-side domain aggregates.

Preferred:

```text
HTTP Handler
     ↓
Query Service
     ↓
Query Repository
     ↓
PostgreSQL
```

A query repository may use SQL specifically optimized for the response shape.

For example:

```go
type ProductListItem struct {
    ID          uuid.UUID
    Name        string
    Slug        string
    Price       Money
    ThumbnailURL string
}
```

Do not load a complete Product aggregate when the API only needs six fields.

---

# 41. Lightweight CQRS

CQRS may be used conceptually without introducing a CQRS framework.

Example:

```text
WRITE
WooCommerce Sync
       ↓
Domain / Repositories
       ↓
PostgreSQL

READ
HTTP API
       ↓
Query Service
       ↓
Query Repository
       ↓
PostgreSQL
```

Do not introduce event sourcing or separate read databases unless the application genuinely requires them.

---

# 42. HTTP Handlers

Handlers should be thin.

Handler responsibility:

```text
parse request
validate request-level input
call service/query
translate result to HTTP response
```

Avoid placing business logic in handlers.

Bad:

```go
func (h *Handler) CreateProduct(c *gin.Context) {
    // 200 lines of business logic
}
```

Preferred:

```go
func (h *Handler) CreateProduct(c *gin.Context) {
    ...
    product, err := h.service.Create(ctx, input)
    ...
}
```

---

# 43. Request Validation

Validate external input at the boundary.

Examples:

```text
required fields
UUID format
pagination limits
allowed enum values
numeric ranges
```

Do not allow invalid input to travel deep into the system before discovering the problem.

---

# 44. Comments

Comments should explain:

```text
why
```

rather than:

```text
what obvious code does
```

Bad:

```go
// Create product
product := &Product{}
```

Useful:

```go
// Slug is Gluzo-owned and must not be overwritten during source synchronization.
```

Keep comments brief.

Do not generate extensive documentation comments unless required by the public API or Go tooling.

---

# 45. Utility Packages

Do not create a generic:

```text
utils/
helpers/
common/
```

package by default.

Before creating a helper, ask:

> Which package owns this behavior?

Put the function there if possible.

Example:

```go
product.ParsePrice()
```

may belong to the product/money domain rather than:

```go
utils.ParsePrice()
```

---

# 46. Generics

Use generics only when they genuinely reduce duplication while preserving readability.

Do not introduce generics simply because Go supports them.

Prefer straightforward code when the generic abstraction is harder to understand than the duplicated code.

---

# 47. Reflection

Avoid reflection unless it is genuinely necessary.

Prefer explicit Go code.

Reflection-heavy designs usually make business logic harder to understand and debug.

---

# 48. Global State

Avoid mutable global state.

Bad:

```go
var db *pgxpool.Pool
var config Config
```

Prefer dependency injection.

Immutable package-level constants are fine:

```go
const DefaultPageSize = 24
```

---

# 49. Configuration

Configuration should be explicit.

Example:

```go
type Config struct {
    DatabaseURL string
    Server      ServerConfig
    WooCommerce WooCommerceConfig
}
```

Avoid reading environment variables throughout business logic.

Load configuration at the application boundary and inject it.

---

# 50. Application Wiring

Dependency creation should happen near application startup.

Conceptually:

```text
Config
  ↓
Database
  ↓
Repositories
  ↓
External Clients
  ↓
Services
  ↓
Handlers
  ↓
Routes
```

This makes dependency relationships visible.

---

# 51. Do Not Over-Engineer

Before introducing:

```text
factory
strategy
builder
mediator
command bus
event bus
generic repository
dependency container
CQRS framework
```

ask:

> Does the current requirement actually need this?

Use design patterns when they solve an actual problem.

Do not use patterns to demonstrate knowledge of patterns.

---

# 52. Refactoring Existing Code

When modifying existing code:

1. Understand current behavior.
2. Identify the smallest safe change.
3. Preserve existing public behavior unless change is required.
4. Follow existing naming.
5. Avoid unrelated refactoring.
6. Avoid changing multiple architectural boundaries at once.

Prefer:

```text
small coherent changes
```

over:

```text
rewrite the module
```

# 53. Production-Grade Does Not Mean Maximum Complexity

Production-grade code means:

```text
clear ownership
correct transactions
safe SQL
explicit dependencies
proper error handling
correct data modeling
predictable behavior
```

It does not automatically mean:

```text
microservices
Kafka
Redis
CQRS frameworks
event sourcing
distributed locks
multiple databases
complex abstractions
```

Use complexity only when justified by a real requirement.

---

# 54. Testing Constraint

For this coding style, do not introduce:

* TDD
* unit-test frameworks
* integration-test frameworks
* testing frameworks
* generated test infrastructure

unless the user explicitly asks for testing.

When verification is required, focus on:

```text
compile correctness
static code reasoning
database/schema inspection
end-to-end flow inspection
manual verification
```

Do not add test infrastructure merely because a generic Go development guide recommends it.

---

# 55. Performance

Do not prematurely optimize.

First ensure:

```text
correctness
data integrity
clear architecture
```

Optimize when there is an identified bottleneck.

However, avoid obvious production mistakes such as:

```text
N+1 queries
unbounded pagination
loading huge datasets unnecessarily
creating a new HTTP client per request
missing database indexes for obvious lookup keys
```

# 56. API Response Design

API responses should represent the API contract, not database tables.

For example:

```json
{
  "data": {
    "id": "...",
    "name": "...",
    "slug": "..."
  }
}
```

Do not expose:

```text
database foreign keys
internal mapping IDs
internal implementation details
```

unless they are intentionally part of the public API.

---

# 57. Read vs Write Models

It is acceptable for write-side domain structures and read-side DTOs to differ.

Example:

```text
Write:
Product aggregate

Read:
ProductListItem
ProductDetail
```

Do not force a single model to serve every purpose.

---

# 58. Idempotency

For synchronization and retryable operations, ask:

> What happens if this operation runs twice?

The result should generally remain correct.

Examples:

```text
Sync product 1001
Sync product 1001
```

should not create:

```text
two Gluzo products
two external mappings
duplicate relationship rows
```

Use database uniqueness constraints as a final protection.

---

# 59. External Synchronization Pattern

For a source integration:

```text
External API
     ↓
Client
     ↓
External Model
     ↓
Mapper
     ↓
Application Service
     ↓
Transaction
     ↓
Internal Domain
     ↓
External Mapping
```

External mapping is the bridge between:

```text
source identity
```

and:

```text
internal identity
```

---

# 60. Source Mapping Rule

If the same external entity is encountered again:

```text
source + external ID
```

must resolve to the same internal entity unless the business explicitly allows remapping.

Do not identify source records solely by:

```text
name
slug
SKU
email
```

when the source provides a stable ID.

---

# 61. Code Style

Prefer standard `gofmt` formatting.

Prefer short functions.

Prefer early returns.

Example:

```go
if err != nil {
    return nil, err
}
```

rather than deeply nested:

```go
if err == nil {
    ...
}
```

Keep control flow easy to follow.

---

# 62. Avoid Clever Go

Do not optimize for cleverness.

Prefer:

```go
for _, item := range items {
    ...
}
```

over obscure functional-style abstractions.

Prefer explicit code when it is easier to understand.

Go code should be boring in a good way.

---

# 63. Documentation

Documentation should explain meaningful architecture and behavior.

Do not generate large documentation files for every function.

For major modules, useful documentation includes:

```text
purpose
responsibilities
data flow
important architectural decisions
```

---

# 64. AI Self-Review Before Finishing

Before declaring a feature complete, inspect:

### Architecture

```text
Is responsibility in the correct package?
Are dependencies explicit?
Did I introduce unnecessary abstractions?
```

### Database

```text
Are constraints correct?
Are transactions correct?
Are queries parameterized?
```

### Errors

```text
Are errors propagated?
Is useful context preserved?
```

### Data

```text
Are IDs correct?
Are external IDs separated?
Is money safe?
Are timestamps correct?
```

### API

```text
Are request/response DTOs appropriate?
Is input validated?
Are internal fields hidden?
```

### Integration

```text
Are external API failures handled?
Is synchronization idempotent?
```

### Existing Code

```text
Did I follow the repository's established conventions?
Did I accidentally redesign unrelated code?
```

---

# 65. Final AI Response Format

When completing a coding task, provide a concise summary:

```text
Implemented:
- ...
- ...
- ...

Files changed:
- ...
- ...

Architecture:
- ...

Important decisions:
- ...

Verification:
- ...

Next step:
- ...
```

Do not provide a huge explanation unless requested.

---

# 66. Golden Rules

The AI must remember these rules above all others:

```text
1. Understand before modifying.
2. Existing repository conventions take priority.
3. Prefer domain-oriented packages.
4. Keep dependencies explicit.
5. Keep interfaces small.
6. Do not create interfaces without a reason.
7. Keep handlers thin.
8. Keep repositories focused on persistence.
9. Keep services focused on business orchestration.
10. Keep mappers free of side effects.
11. Use transactions around business operations.
12. Keep transaction-scoped repositories on the same transaction.
13. Never use float64 for money.
14. Never confuse external IDs with internal IDs.
15. Make synchronization idempotent where required.
16. Use database constraints for data integrity.
17. Parameterize SQL.
18. Whitelist dynamic SQL elements.
19. Do not over-engineer.
20. Prefer boring, explicit, idiomatic Go.
21. Make the smallest coherent change.
22. Do not redesign unrelated code.
23. Verify the complete flow, not only individual files.
24. Preserve established architectural decisions.
25. Production-grade means reliable and understandable, not unnecessarily complex.
```

---

# 67. Adaptation Rule

This skill is a **default engineering style**, not a rigid framework.

When an existing repository differs:

```text
Existing coherent architecture
        >
This skill's default preference
```

The AI must adapt.

However, when introducing a genuinely new module into an existing repository, use this skill to establish the new module's architecture while matching the surrounding repository conventions.

The goal is consistency, not architectural uniformity at all costs.

---

# 94. Expected Engineering Style

The final Go code should feel:

```text
Idiomatic Go
+
Explicit architecture
+
Domain-oriented
+
Production-safe
+
Minimal abstraction
+
Clear dependencies
+
Strong data integrity
+
Easy to maintain
```

The AI should write code as an experienced Go engineer would write it for a real production system—not as a demonstration of every Go pattern available.
