# 93. Configurable Application Logging

The application must support **configurable file-based logging**.

The goal is to provide a logging experience similar in spirit to Laravel's configurable logging system while keeping the Go implementation simple, explicit, and idiomatic.

The logging system should be:

```text
Configurable
File-based
Structured
Environment-aware
Retention-aware
Simple
```

Do not introduce a large logging/observability platform unless explicitly required.

---

# 94. Logging Philosophy

Logs are part of the application's operational behavior.

The application should make it possible to configure:

```text
log level
log channel/type
log directory
log file naming
log retention
console output
file output
application environment
```

The exact implementation can differ between projects, but the configuration model should remain explicit.

The goal is similar to:

```text
Laravel
    ↓
logging.php
    ↓
channels
    ↓
daily / single / stack
    ↓
path
    ↓
level
    ↓
retention
```

but implemented using idiomatic Go.

---

# 95. Logging Configuration

Logging configuration should be represented by a dedicated configuration structure.

Example:

```go
type LoggingConfig struct {
    Level           string
    Directory       string
    FileName        string
    MaxAgeDays      int
    Console         bool
    File            bool
}
```

For more flexible applications, use channel-based configuration:

```go
type LoggingConfig struct {
    DefaultChannel string
    Channels       map[string]LogChannelConfig
}

type LogChannelConfig struct {
    Type       string
    Level      string
    Directory  string
    FileName   string
    Retention  int
    Console    bool
}
```

The second approach is preferred when the application genuinely needs multiple log channels.

Do not introduce channel configuration merely for the sake of abstraction.

---

# 96. Environment Configuration

Logging configuration should be configurable through application configuration/environment variables.

Example:

```text
LOG_CHANNEL=daily
LOG_LEVEL=info
LOG_DIRECTORY=storage/logs
LOG_RETENTION_DAYS=30
LOG_CONSOLE=true
LOG_FILE=true
```

The exact environment variable names may follow the existing project's conventions.

Do not read environment variables directly throughout the application.

Instead:

```text
Environment
    ↓
Config Loader
    ↓
LoggingConfig
    ↓
Logger Initialization
    ↓
Application
```

---

# 97. Daily Log Files

The default file-based logging strategy should support daily log files.

Example:

```text
storage/logs/
├── app-2026-09-27.log
├── app-2026-09-28.log
├── app-2026-09-29.log
└── ...
```

This provides predictable operational behavior and makes logs easy to inspect.

The filename pattern should be configurable where practical.

---

# 98. Retention

Log retention must be configurable.

Example:

```text
LOG_RETENTION_DAYS=30
```

If configured for 30 days:

```text
today
  ↓
retain last 30 days
  ↓
delete older log files
```

Retention should not require manual cleanup.

The logging system should perform retention cleanup safely.

Do not delete arbitrary files from the configured directory.

Only delete files that match the configured logging file pattern and are older than the configured retention period.

---

# 99. Log Directory

The log directory must be configurable.

Examples:

```text
storage/logs
/var/log/gluzo
/var/www/app/storage/logs
```

Do not hardcode:

```text
/var/log/app
```

inside application code.

The default may be:

```text
storage/logs
```

for local development, but production configuration should be able to override it.

---

# 100. Log Channels

Support the concept of channels when there is a real need for different categories of logs.

For example:

```text
application
database
integration
woocommerce
auth
```

Possible configuration:

```text
LOG_CHANNEL=application
```

or:

```text
channels:
    application
    integration
    auth
```

However, do not create dozens of channels.

A channel should exist because someone has a reason to independently:

```text
inspect
filter
retain
route
```

that category of logs.

---

# 101. Example Channel Configuration

A production application could conceptually support:

```yaml
logging:
  default_channel: application

  channels:
    application:
      type: daily
      level: info
      directory: storage/logs
      file_name: app
      retention_days: 30

    integration:
      type: daily
      level: info
      directory: storage/logs/integration
      file_name: integration
      retention_days: 30

    auth:
      type: daily
      level: warning
      directory: storage/logs/auth
      file_name: auth
      retention_days: 14
```

This is only an example.

The actual configuration format should follow the project's existing configuration system.

---

# 102. Log Levels

Support standard log levels.

At minimum:

```text
debug
info
warn
error
```

If the selected logging library supports additional levels, they may be used where appropriate.

The configured level determines which messages are written.

Example:

```text
LOG_LEVEL=info
```

means:

```text
debug → ignored
info  → written
warn  → written
error → written
```

---

# 103. Structured Logs

Prefer structured logs for application code.

Example:

```json
{
  "level": "error",
  "message": "sync product failed",
  "product_id": "123",
  "source": "woocommerce",
  "error": "connection timeout"
}
```

Use fields instead of constructing large human-readable strings containing all metadata.

Prefer:

```go
logger.Error(
    "sync product failed",
    "product_id", productID,
    "source", "woocommerce",
    "error", err,
)
```

over:

```go
logger.Error(
    fmt.Sprintf(
        "product %s failed while syncing from WooCommerce: %v",
        productID,
        err,
    ),
)
```

---

# 104. Go Logging Library

Prefer the standard library where it satisfies the application's needs.

For modern Go applications:

```go
log/slog
```

is the preferred default structured logging API.

Do not introduce a third-party logging framework unless there is a concrete requirement that the standard library does not satisfy.

The goal is:

```text
simple
standard
structured
configurable
```

---

# 105. Logger Initialization

Logger creation should happen during application startup.

Conceptually:

```text
Config
  ↓
LoggingConfig
  ↓
Logger Factory
  ↓
Configured Logger
  ↓
Application Dependencies
```

Do not create a new logger repeatedly throughout the application.

---

# 106. Logger Dependency Injection

Where practical, inject the logger into components that need application logging.

Example:

```go
type Service struct {
    repository Repository
    logger     *slog.Logger
}
```

Constructor:

```go
func NewService(
    repository Repository,
    logger *slog.Logger,
) *Service {
    return &Service{
        repository: repository,
        logger:     logger,
    }
}
```

Do not create a logger inside every service.

---

# 107. Avoid Global Logger Abuse

A package-level logger may be acceptable for very small applications, but dependency injection is preferred for larger applications.

Avoid:

```go
var Logger = slog.Default()
```

being imported and used everywhere if it makes dependencies invisible.

The preferred architecture is:

```text
Application
    ↓
Logger
    ↓
Service
    ↓
Repository
```

However, do not create excessive ceremony around logger injection for tiny utility functions that do not need logging.

---

# 108. Console + File Logging

Support separate destinations when useful.

Development:

```text
Console = true
File    = true
```

Production:

```text
Console = optional
File    = true
```

The exact configuration depends on deployment requirements.

Do not assume console logging should always be disabled in production.

---

# 109. Application Logs vs Integration Logs

For systems with third-party integrations, integration logs may deserve a separate channel.

Example:

```text
storage/logs/
├── app/
│   └── app-2026-09-27.log
└── integration/
    └── integration-2026-09-27.log
```

This is particularly useful for:

```text
WooCommerce
EasyEcom
UniCommerce
Vinculum
payment gateways
shipping providers
```

But do not duplicate every application log into every channel.

---

# 110. Integration Logging

Integration logs should capture useful operational context.

For example:

```text
source
operation
external ID
HTTP status
duration if useful
success/failure
error
```

Example:

```json
{
  "level": "error",
  "channel": "integration",
  "integration": "woocommerce",
  "operation": "get_product",
  "external_id": 1234,
  "status": 500,
  "message": "WooCommerce API request failed"
}
```

Do not automatically log complete request/response bodies.

---

# 111. Sensitive Data

Never log:

```text
passwords
JWT secrets
API secrets
consumer secrets
OTP values
credit card data
authentication tokens
```

Be careful with:

```text
Authorization headers
cookies
personal information
payment information
```

If request/response logging is required, redact sensitive fields.

---

# 112. Logging Errors

Logging and returning an error are separate responsibilities.

For example:

```go
product, err := service.Sync(ctx, id)
if err != nil {
    logger.Error(
        "product sync failed",
        "product_id", id,
        "error", err,
    )

    return err
}
```

Do not swallow an error simply because it was logged.

Bad:

```go
if err != nil {
    logger.Error("failed", "error", err)
    return nil
}
```

unless the operation is intentionally non-fatal.

---

# 113. Avoid Duplicate Logging

Avoid logging the same error at every layer.

Bad:

```text
Repository:
  database error

Service:
  repository failed

Handler:
  service failed

Global middleware:
  request failed
```

This can produce four log entries for one failure.

Prefer:

```text
lower layer
    ↓
return contextual error

boundary / appropriate owner
    ↓
log the failure
```

The exact boundary depends on the application.

---

# 114. Correlation IDs

When the application already has request/correlation IDs, include them in structured logs.

Example:

```json
{
  "level": "error",
  "request_id": "abc-123",
  "product_id": "uuid",
  "message": "product sync failed"
}
```

Do not introduce a complicated distributed tracing system merely to obtain correlation IDs.

A simple request ID is sufficient when that is all the application requires.

---

# 115. Log Rotation

Daily rotation should be supported when the application requires Laravel-style daily logs.

Rotation should be deterministic:

```text
app-2026-09-27.log
app-2026-09-28.log
app-2026-09-29.log
```

If an external rotation mechanism is already provided by the infrastructure, the application should respect that architecture rather than implementing duplicate rotation.

---

# 116. Retention Cleanup

Retention cleanup should:

1. identify files belonging to the configured logger
2. determine file age
3. remove files older than retention
4. ignore unrelated files
5. fail safely

Example:

```text
retention = 30 days

app-2026-08-20.log → delete
app-2026-09-01.log → retain
app-2026-09-27.log → retain
```

Do not recursively delete arbitrary files under the log directory.

---

# 117. Logging Configuration Should Be Centralized

The application should have one clear location where logging configuration is defined.

For example:

```text
internal/config/
    config.go

internal/logging/
    logger.go
    config.go
```

Possible structure:

```text
internal/
├── config/
│   └── config.go
├── logging/
│   ├── config.go
│   ├── logger.go
│   └── rotation.go
└── ...
```

Do not spread logging initialization across unrelated packages.

---

# 118. Logging Package Responsibility

The logging package should own:

```text
logger creation
handler configuration
file creation
rotation
retention
channel configuration
```

It should not own:

```text
business logic
database logic
HTTP handling
domain decisions
```

---

# 119. Logging Is Infrastructure, Not Business Logic

Business code should say:

```go
logger.Info(
    "product synchronized",
    "product_id", productID,
)
```

It should not know:

```text
which directory
which filename
how rotation works
how retention works
which file descriptor is open
```

Those are logging infrastructure concerns.

---

# 120. Laravel-Inspired Configuration, Go Implementation

The desired mental model is:

```text
Laravel
----------------------------
logging configuration
channels
daily files
retention
directories
levels


Go
----------------------------
logging configuration
channels
daily files
retention
directories
levels
        ↓
implemented with idiomatic Go
```

Do not attempt to clone Laravel's internal logging architecture.

Copy the **configuration philosophy**, not the framework implementation.

---

# 121. Default Logging Configuration

A reasonable default for a new application:

```text
channel: daily
level: info
directory: storage/logs
retention: 30 days
console: true
file: true
```

These are defaults only.

Production configuration must be able to override them.

---

# 122. Fresh Repository Logging

For a new Go repository, establish logging early because application components should not invent their own logging approach.

Recommended startup sequence:

```text
Load Config
    ↓
Initialize Logger
    ↓
Initialize Database
    ↓
Initialize External Clients
    ↓
Initialize Repositories
    ↓
Initialize Services
    ↓
Initialize Handlers
    ↓
Start Application
```

---

# 123. Existing Repository Logging

When adding logging to an existing repository:

1. Inspect existing logging.
2. Preserve existing logger APIs where reasonable.
3. Determine whether configuration already exists.
4. Extend rather than duplicate the logging system.
5. Introduce file rotation/retention only if missing and required.
6. Avoid changing unrelated application behavior.

Never create:

```text
old logger
+
new logger
```

without a clear migration strategy.

---

# 124. Logging Verification

When implementing or modifying logging, verify:

* logger initializes successfully
* configured directory is created if appropriate
* file logging works
* console logging follows configuration
* log level filtering works
* daily files are created
* retention removes only eligible files
* channel configuration works
* application continues to function if a log write encounters an expected recoverable issue
* sensitive information is not written
* concurrent application logging does not corrupt files

---

# 125. Updated Golden Rule

The previous golden rule:

> Go code should be boring in a good way.

Now means:

```text
Go code should be boring in a good way.

The architecture should be explicit.

The logging should be configurable.

The configuration should be boring.

The application should not care where its logs are stored.
```

The developer should be able to change:

```text
log level
log directory
log channel
log retention
console/file output
```

through configuration without modifying business code.
