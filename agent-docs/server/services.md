# Backend Services

This document covers all services in `server/services/` and `server/auth/`.

---

## Auth Service (`server/auth/`)

**File:** `server/auth/auth.go`

The Auth service handles session validation and read-token checking.

```go
type Auth struct {
    config      *config.Configuration
    store       store.Store
    permissions permissions.PermissionsService
}

func New(config, store, permissions) *Auth
```

### Methods

| Method | Description |
|--------|-------------|
| `GetSession(token string) (*model.Session, error)` | Validate session token. Auto-refreshes if within refresh window. Returns `ErrInvalidToken` if not found or expired. |
| `IsValidReadToken(boardID, readToken string) (bool, error)` | Check if `readToken` matches the stored sharing token for `boardID`. Returns `false` if public sharing is disabled in config. |
| `DoesUserHaveTeamAccess(userID, teamID string) bool` | Check if user is a member of the team. |

### Session Refresh Logic

Sessions are refreshed when:
- The session's `UpdateAt` is older than `config.SessionRefreshTime`
- The session has not yet expired (`UpdateAt + config.SessionExpireTime > now`)

---

## Config Service (`server/services/config/`)

**File:** `server/services/config/config.go`

### Configuration struct

```go
type Configuration struct {
    // Server
    ServerRoot   string `json:"serverRoot"`
    Port         int    `json:"port"`
    WebPath      string `json:"webPath"`
    LocalOnly    bool   `json:"localOnly"`

    // Database
    DBType             string `json:"dbtype"`
    DBConfigString     string `json:"dbconfig"`
    DBPingAttempts     int    `json:"dbPingAttempts"`
    DBTablePrefix      string `json:"dbtableprefix"`

    // Files
    FilesDriver  string       `json:"filesdriver"` // "local" or "amazons3"
    FilesPath    string       `json:"filespath"`
    FilesS3Config AmazonS3Config
    MaxFileSize  int64        `json:"maxFileSize"`

    // Security
    UseSSL            bool   `json:"useSSL"`
    SecureCookie      bool   `json:"secureCookie"`
    Secret            string `json:"secret"`
    SessionExpireTime int64  `json:"session_expire_time"` // seconds
    SessionRefreshTime int64 `json:"session_refresh_time"` // seconds
    PasswordMinLength int    `json:"passwordMinLength"`
    PasswordMaxLength int    `json:"passwordMaxLength"`

    // Auth
    AuthMode string `json:"authMode"` // "native" or "mattermost"

    // Features
    EnablePublicSharedBoards bool   `json:"enablePublicSharedBoards"`
    EnableDataRetention      bool   `json:"enableDataRetention"`
    DataRetentionDays        int    `json:"dataRetentionDays"`

    // Telemetry
    Telemetry         bool   `json:"telemetry"`
    TelemetryID       string `json:"telemetryid"`
    PrometheusAddress string `json:"prometheusaddress"`

    // Logging
    LoggingCfgFile string `json:"logging_cfg_file"`
    LoggingCfgJSON string `json:"logging_cfg_json"`

    // Audit
    AuditCfgFile string `json:"audit_cfg_file"`
    AuditCfgJSON string `json:"audit_cfg_json"`

    // Notifications
    NotifyFreqCardSeconds  float64 `json:"notify_freq_card_seconds"`
    NotifyFreqBoardSeconds float64 `json:"notify_freq_board_seconds"`

    // UI
    TeammateNameDisplay string `json:"teammate_name_display"`
    ShowEmailAddress    bool   `json:"show_email_address"`
    ShowFullName        bool   `json:"show_full_name"`

    // Webhooks
    WebhookUpdate []string `json:"webhook_update"`

    // Feature flags
    FeatureFlags map[string]string `json:"featureFlags"`

    // Admin
    EnableLocalMode          bool   `json:"enableLocalMode"`
    LocalModeSocketLocation  string `json:"localModeSocketLocation"`

    // Single user
    SingleUserToken string `json:"-"`
}
```

### Defaults

| Setting | Default Value |
|---------|---------------|
| `ServerRoot` | `http://localhost:8000` |
| `Port` | `8000` |
| `DBType` | `sqlite3` |
| `DBConfigString` | `./focalboard.db` |
| `WebPath` | `./pack` |
| `FilesPath` | `./files` |
| `FilesDriver` | `local` |
| `SessionExpireTime` | `2592000` (30 days) |
| `SessionRefreshTime` | `300` (5 minutes) |

### Loading

```go
ReadConfigFile(configFilePath string) (*Configuration, error)
```

1. Reads JSON file with `viper`
2. Overrides from environment variables (prefix: `FOCALBOARD_`)
3. Fills defaults for unset fields

---

## S3 Config (`AmazonS3Config`)

```go
type AmazonS3Config struct {
    AccessKeyID     string
    SecretAccessKey string
    Bucket          string
    PathPrefix      string
    Region          string
    Endpoint        string
    SSL             bool
    SignV2          bool
    SSE             bool
    Trace           bool
    Timeout         int64
}
```

---

## Permissions Service (`server/services/permissions/`)

**File:** `server/services/permissions/permissions.go`

```go
type PermissionsService interface {
    HasPermissionTo(userID string, permission *mm_model.Permission) bool
    HasPermissionToTeam(userID, teamID string, permission *mm_model.Permission) bool
    HasPermissionToChannel(userID, chID string, permission *mm_model.Permission) bool
    HasPermissionToBoard(userID, boardID string, permission *mm_model.Permission) bool
}
```

### Implementations

#### Local Permissions (`localpermissions/`)

Used in standalone mode. Board access based on:
1. Is the user a board member with the required role?
2. Is the board open (`Type: "O"`) and the permission is `ViewBoard`?
3. Is the user a system admin?

Role hierarchy: `admin > editor > commenter > viewer`

Each role grants all permissions of lower roles.

#### Mattermost Permissions (`mmpermissions/`)

Used in plugin mode. Delegates to the Mattermost plugin API for team/channel/system permissions. Board permissions still managed by Focalboard's own membership table, but inherit from Mattermost channel membership.

---

## Metrics Service (`server/services/metrics/`)

**Files:** `service.go`, `metrics.go`

### MetricsService (HTTP Server)

```go
type Service struct {
    *http.Server
}
func NewMetricsServer(address string, metrics *Metrics, logger) *Service
func (s *Service) Run() error
func (s *Service) Shutdown() error
```

Serves Prometheus `/metrics` endpoint at `config.PrometheusAddress`.

### Metrics (Counters/Gauges)

```go
type Metrics struct {
    registry *prometheus.Registry
    // Counters
    BlocksPatched     prometheus.Counter
    BlocksInserted    prometheus.Counter
    Logins            prometheus.Counter
    LoginFailures     prometheus.Counter
    // Gauges
    BlockCount        *prometheus.GaugeVec  // keyed by type
    BoardCount        prometheus.Gauge
    TeamCount         prometheus.Gauge
}

// Methods
func (m *Metrics) ObserveBlockCount(blockType string, count int64)
func (m *Metrics) ObserveBoardCount(count int64)
func (m *Metrics) ObserveTeamCount(count int64)
func (m *Metrics) IncrementBlocksPatched(count int)
func (m *Metrics) IncrementBlocksInserted(count int)
func (m *Metrics) IncrementLogins()
func (m *Metrics) IncrementLoginFailures()
```

### InstanceInfo

```go
type InstanceInfo struct {
    Version        string
    BuildNum       string
    Edition        string
    InstallationID string
}
```

---

## Notification Service (`server/services/notify/`)

**File:** `server/services/notify/service.go`

### Service

```go
type Service struct {
    mux      sync.RWMutex
    backends []Backend
    logger   mlog.LoggerIFace
}

func New(logger, backends ...Backend) (*Service, error)
func (s *Service) Start() error
func (s *Service) Shutdown() error
func (s *Service) BlockChanged(evt BlockChangeEvent) error
```

### Backend Interface

```go
type Backend interface {
    Start() error
    ShutDown() error
    BlockChanged(evt BlockChangeEvent) error
    Name() string
}
```

### BlockChangeEvent

```go
type BlockChangeEvent struct {
    Action       Action         // "add", "update", "delete"
    TeamID       string
    Board        *model.Board
    Card         *model.Block   // Parent card block
    BlockChanged *model.Block   // The changed block
    BlockOld     *model.Block   // Previous state
    ModifiedBy   *model.BoardMember
}
```

### Built-in Backends

#### `notifylogger`
- Logs every block change event at debug level
- Always included, cannot be disabled

#### `notifysubscriptions` (plugin mode)
- Finds all subscribers of the changed block
- Schedules `NotificationHint` in DB for deferred delivery
- A separate goroutine polls for pending hints and sends them

#### `notifymentions` (plugin mode)
- Parses block `Title` and content for `@username` mentions
- Sends direct messages via Mattermost plugin API

#### `plugindelivery` (plugin mode)
- Delivers formatted notification messages to Mattermost DMs or channels
- Uses `PostMessage` / `SendMessage` from the store

---

## Webhook Service (`server/services/webhook/`)

**File:** `server/services/webhook/webhook.go`

```go
type Client struct {
    config *config.Configuration
    logger mlog.LoggerIFace
}

func NewClient(config, logger) *Client
func (wh *Client) NotifyUpdate(block *model.Block)
```

### Behavior

- Reads `config.WebhookUpdate` (list of URLs)
- For each changed block, sends an HTTP POST to each configured URL
- Request body: JSON of the `Block`
- Content-Type: `application/json`
- Fire-and-forget (errors are logged, not returned)

---

## Telemetry Service (`server/services/telemetry/`)

**File:** `server/services/telemetry/service.go`

```go
type Service struct {
    telemetryID string
    logger      mlog.LoggerIFace
    trackers    map[string]TrackerFunc
}

type TrackerFunc func() (Tracker, error)
type Tracker = map[string]interface{}

func New(telemetryID, logger) *Service
func (s *Service) RegisterTracker(name string, fn TrackerFunc)
func (s *Service) RunTelemetryJob(firstRun int64)
func (s *Service) Shutdown() error
```

### Registered Trackers

| Tracker Name | Data |
|---|---|
| `server` | version, build_number, build_hash, edition, OS, server_id |
| `config` | serverRoot default, port default, useSSL, dbType, single_user, public boards |
| `activity` | registered_users, daily/weekly/monthly active users |
| `blocks` | count per block type |
| `boards` | total boards |
| `teams` | total teams |

Data is sent to Mattermost's telemetry endpoint. All data is anonymous (no content, just counts and flags).

---

## Audit Service (`server/services/audit/`)

**File:** `server/services/audit/audit.go`

```go
type Audit struct {
    // internal implementation
}

func NewAudit() (*Audit, error)
func (a *Audit) Configure(cfgFile, cfgJSON string) error
func (a *Audit) LogRecord(level, rec AuditRecord) error
func (a *Audit) Shutdown() error
```

### AuditRecord

```go
type AuditRecord struct {
    EventName string
    Status    string // "attempt", "success", "fail"
    UserID    string
    SessionID string
    Client    string
    IPAddress string
    Meta      []AuditField
}
```

Output targets are configured via `AuditCfgFile` or `AuditCfgJSON`. Supports file, syslog, and TCP targets.

---

## Scheduler (`server/services/scheduler/`)

**File:** `server/services/scheduler/scheduler.go`

```go
type ScheduledTask struct {
    Name     string
    Interval time.Duration
    ticker   *time.Ticker
    quit     chan struct{}
}

func CreateRecurringTask(name string, fn func(), interval time.Duration) *ScheduledTask
func (t *ScheduledTask) Cancel()
```

Simple ticker-based task runner. Used for:
- `cleanUpSessions` — every 10 minutes
- `updateMetrics` — every 15 minutes
