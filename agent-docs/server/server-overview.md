# Server Overview — Startup, Lifecycle, and Structure

## Entry Point

**File:** `server/main/main.go`

The `main` package is the binary entry point for the standalone server. It:

1. Parses command-line flags:
   - `-config` — Path to the JSON config file (default `config.json`)
   - `-port` — Override the port from config
   - `-single-user` — Enable single-user mode (no login required)
   - `-dbtype` — Override the database type (`sqlite3`, `postgres`, `mysql`)
   - `-dbconfig` — Override the database connection string
   - `-monitorpid` — PID to monitor; shut down server if that process dies
2. Loads the configuration via `config.ReadConfigFile()`.
3. Initializes the Mattermost `mlog` logger from the config.
4. Logs version/build/OS info.
5. Creates the database store (`server.NewStore()`).
6. Creates the fully initialized `server.Server` via `server.New()`.
7. Calls `server.Start()` and blocks until a shutdown signal (SIGTERM / SIGINT) is received.
8. Calls `server.Shutdown()` for graceful teardown.

---

## Server Struct

**File:** `server/server/server.go`

```go
type Server struct {
    config                 *config.Configuration
    wsAdapter              ws.Adapter             // WebSocket broadcaster
    webServer              *web.Server            // HTTP server (gorilla/mux)
    store                  store.Store            // Database abstraction
    filesBackend           filestore.FileBackend  // File storage
    telemetry              *telemetry.Service     // Usage analytics
    logger                 mlog.LoggerIFace
    cleanUpSessionsTask    *scheduler.ScheduledTask
    metricsServer          *metrics.Service       // Prometheus HTTP server
    metricsService         *metrics.Metrics       // Prometheus counters/gauges
    metricsUpdaterTask     *scheduler.ScheduledTask
    auditService           *audit.Audit
    notificationService    *notify.Service
    servicesStartStopMutex sync.Mutex

    localRouter     *mux.Router    // Admin API router
    localModeServer *http.Server   // Unix socket admin server
    api             *api.API       // HTTP API handler
    app             *app.App       // Business logic layer
}
```

---

## `server.New()` — Initialization Sequence

**File:** `server/server/server.go` — `New(params Params) (*Server, error)`

Steps executed in order:

| Step | What happens |
|------|-------------|
| 1 | `params.CheckValid()` — validates required fields |
| 2 | `auth.New(cfg, store, permissions)` — creates the authenticator |
| 3 | If no `WSAdapter` provided, `ws.NewServer(...)` — standalone WebSocket server |
| 4 | Configure and create file storage backend (local or S3) via Mattermost `filestore` |
| 5 | `webhook.NewClient(cfg)` — outbound webhook dispatcher |
| 6 | `metrics.NewMetrics(instanceInfo)` — Prometheus metrics |
| 7 | `audit.NewAudit()` + `Configure(...)` — audit logging service |
| 8 | `initNotificationService(backends, logger)` — wraps all notify backends; always includes `notifylogger` |
| 9 | `app.New(cfg, wsAdapter, appServices)` — business logic with all services injected |
| 10 | `api.NewAPI(app, singleUserToken, authMode, permissions, logger, audit)` — HTTP handler layer |
| 11 | Register admin routes on a separate `localRouter` |
| 12 | `app.GetRootTeam()` — ensures the root team (ID `"0"`) exists in the database |
| 13 | `web.NewServer(webPath, serverRoot, port, useSSL, localOnly)` — HTTP server |
| 14 | Register WebSocket adapter routes on the web server (if adapter is `RoutedService`) |
| 15 | Register API routes on the web server |
| 16 | Read `TelemetryID` system setting; create if missing |
| 17 | `initTelemetry(opts)` — registers telemetry trackers |
| 18 | Build and return the `Server` struct |
| 19 | `server.initHandlers()` — sets up OS signal handlers (`SIGTERM`, `SIGINT`, `SIGHUP`) |

---

## `server.Start()`

```
Start()
├─ webServer.Start()               → HTTP listener on configured port
├─ startLocalModeServer()          → Unix socket admin server (if EnableLocalMode)
├─ scheduler.CreateRecurringTask(  → Session cleanup every 10 minutes
│     "cleanUpSessions", fn, 10m)     (skipped in Mattermost auth mode)
├─ scheduler.CreateRecurringTask(  → Metrics update every 15 minutes
│     "updateMetrics", fn, 15m)
│     Collects: block counts, board count, team count
└─ telemetry.RunTelemetryJob()     → If Telemetry enabled in config
└─ metricsServer.Run()             → Prometheus HTTP server (if PrometheusAddress set)
```

---

## `server.Shutdown()`

```
Shutdown()
├─ webServer.Shutdown()            → Graceful HTTP server shutdown
├─ stopLocalModeServer()           → Close Unix socket server
├─ cleanUpSessionsTask.Cancel()    → Stop session cleaner
├─ metricsUpdaterTask.Cancel()     → Stop metrics updater
├─ telemetry.Shutdown()            → Flush analytics
├─ auditService.Shutdown()         → Flush audit logs
├─ notificationService.Shutdown()  → Shutdown notification backends
├─ app.Shutdown()                  → Stop block change notifier queue
└─ store.Shutdown()                → Close database connections
```

---

## `server.NewStore()` — Database Connection

Creates a new SQL store:
1. `sql.Open(dbType, connectionString)` — opens driver connection
2. `sqlDB.Ping()` — verifies connectivity (retries if `DBPingAttempts > 0`)
3. `sqlstore.New(params)` — runs schema migrations and returns the store

---

## Background Tasks

| Task | Frequency | Purpose |
|------|-----------|---------|
| `cleanUpSessions` | Every 10 minutes | Deletes expired sessions from the database. Expiry = max(31 days, `config.SessionExpireTime`). Disabled when using Mattermost auth. |
| `updateMetrics` | Every 15 minutes | Queries block/board/team counts and updates Prometheus gauges. |
| `telemetryJob` | Configurable | Sends anonymized usage stats to Mattermost telemetry service. |

---

## Telemetry Trackers

Registered via `initTelemetry()`:

| Tracker | Data Collected |
|---------|----------------|
| `server` | version, build_number, build_hash, edition, OS, server_id |
| `config` | serverRoot (default?), port (default?), useSSL, dbType, single_user, allow_public_shared_boards |
| `activity` | registered_users, daily/weekly/monthly active users |
| `blocks` | count per block type (card, view, text, image, etc.) |
| `boards` | total board count |
| `teams` | total team count |

---

## Local Mode (Admin API via Unix Socket)

When `config.EnableLocalMode = true`, the server starts a second HTTP server listening on a Unix domain socket (default path: `config.LocalModeSocketLocation`). This server exposes admin-only routes (password reset, etc.) accessible only from the local machine. File permissions on the socket are set to `0600`.

---

## Params Struct

```go
type Params struct {
    Cfg                 *config.Configuration
    SingleUserToken     string
    DBStore             store.Store
    Logger              mlog.LoggerIFace
    ServerID            string
    WSAdapter           ws.Adapter         // optional; created if nil
    NotifyBackends      []notify.Backend   // optional additional notify backends
    PermissionsService  permissions.PermissionsService
    ServicesAPI         app.servicesAPI    // Mattermost plugin API (optional)
}
```

---

## Web Server (`web.Server`)

**File:** `server/web/server.go`

Wraps `net/http` and `gorilla/mux`:
- Serves the bundled React webapp from `config.WebPath`
- Falls back to `index.html` for any unrecognized path (SPA routing)
- Registers `RoutedService` handlers (API, WebSocket) with their own sub-routers
- Supports TLS via `config.UseSSL`

---

## Scheduler (`services/scheduler`)

**File:** `server/services/scheduler/`

Simple recurring task abstraction:
- `CreateRecurringTask(name, fn, interval)` — starts a goroutine that runs `fn` every `interval`
- `task.Cancel()` — stops the goroutine
- Used for session cleanup and metrics updates
