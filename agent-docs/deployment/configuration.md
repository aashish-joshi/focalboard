# Configuration Reference

## Overview

Focalboard is configured via a JSON file (default: `config.json`). Every field can be overridden by an environment variable with the prefix `FOCALBOARD_` (e.g., `FOCALBOARD_PORT=8080`).

---

## Full Configuration Reference

```json
{
    "serverRoot": "http://localhost:8000",
    "port": 8000,
    "dbtype": "sqlite3",
    "dbconfig": "./focalboard.db",
    "dbPingAttempts": 0,
    "dbtableprefix": "",
    "useSSL": false,
    "secureCookie": false,
    "webPath": "./pack",
    "filesdriver": "local",
    "filespath": "./files",
    "maxFileSize": 0,
    "telemetry": true,
    "telemetryid": "",
    "prometheusaddress": "",
    "session_expire_time": 2592000,
    "session_refresh_time": 300,
    "localOnly": false,
    "enableLocalMode": false,
    "localModeSocketLocation": "/var/tmp/focalboard_local.socket",
    "authMode": "native",
    "enablePublicSharedBoards": false,
    "logging_cfg_file": "",
    "logging_cfg_json": "",
    "audit_cfg_file": "",
    "audit_cfg_json": "",
    "enableDataRetention": false,
    "dataRetentionDays": 365,
    "notifyFreqCardSeconds": 120,
    "notifyFreqBoardSeconds": 86400,
    "secret": "",
    "passwordMinLength": 0,
    "passwordMaxLength": 0,
    "webhook_update": [],
    "featureFlags": {},
    "filesS3Config": {
        "accessKeyId": "",
        "secretAccessKey": "",
        "bucket": "",
        "pathPrefix": "",
        "region": "",
        "endpoint": "",
        "ssl": false,
        "signV2": false,
        "sse": false,
        "trace": false,
        "timeout": 0
    }
}
```

---

## Server Settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `serverRoot` | string | `http://localhost:8000` | Public-facing root URL. Used in links and telemetry. |
| `port` | int | `8000` | Port to listen on |
| `webPath` | string | `./pack` | Path to built React app static files |
| `useSSL` | bool | `false` | Enable HTTPS (requires SSL certificate) |
| `secureCookie` | bool | `false` | Set Secure flag on session cookies |
| `localOnly` | bool | `false` | Bind only to `localhost` (not all interfaces) |
| `enableLocalMode` | bool | `false` | Enable admin API on Unix socket |
| `localModeSocketLocation` | string | `/var/tmp/focalboard_local.socket` | Path to Unix socket for local mode |

---

## Database Settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `dbtype` | string | `sqlite3` | Database driver: `sqlite3`, `postgres`, `mysql` |
| `dbconfig` | string | `./focalboard.db` | Database connection string |
| `dbPingAttempts` | int | `0` | Number of connection retry attempts (0 = infinite) |
| `dbtableprefix` | string | `""` | Table name prefix (e.g., `fb_`) |

### Connection String Examples

**SQLite:**
```
./focalboard.db
```

**PostgreSQL:**
```
postgres://user:password@localhost/focalboard?sslmode=disable
```

**MySQL:**
```
user:password@tcp(localhost:3306)/focalboard
```

---

## File Storage Settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `filesdriver` | string | `local` | Storage driver: `local` or `amazons3` |
| `filespath` | string | `./files` | Local files directory (when `filesdriver=local`) |
| `maxFileSize` | int64 | `0` | Max upload size in bytes (0 = unlimited) |

### S3 Configuration (`filesS3Config`)

| Field | Type | Description |
|-------|------|-------------|
| `accessKeyId` | string | AWS Access Key ID |
| `secretAccessKey` | string | AWS Secret Access Key |
| `bucket` | string | S3 bucket name |
| `pathPrefix` | string | Path prefix within the bucket |
| `region` | string | AWS region (e.g., `us-east-1`) |
| `endpoint` | string | Custom endpoint URL (for non-AWS S3 like MinIO) |
| `ssl` | bool | Use HTTPS for S3 requests |
| `signV2` | bool | Use AWS Signature V2 (legacy) |
| `sse` | bool | Enable server-side encryption |
| `trace` | bool | Enable request tracing |
| `timeout` | int64 | Request timeout in milliseconds |

---

## Authentication Settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `authMode` | string | `native` | Auth mode: `native` or `mattermost` |
| `secret` | string | `""` | Secret for JWT token signing |
| `session_expire_time` | int64 | `2592000` | Session expiry in seconds (default: 30 days) |
| `session_refresh_time` | int64 | `300` | Session refresh interval in seconds (default: 5 minutes) |
| `passwordMinLength` | int | `0` | Minimum password length (0 = no minimum) |
| `passwordMaxLength` | int | `0` | Maximum password length (0 = no maximum) |

---

## Feature Settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enablePublicSharedBoards` | bool | `false` | Allow boards to be shared publicly |
| `enableDataRetention` | bool | `false` | Enable automatic data retention/purging |
| `dataRetentionDays` | int | `365` | Days to retain data when retention is enabled |

---

## Notification Settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `notifyFreqCardSeconds` | float64 | `120` | Delay (seconds) before sending card change notification |
| `notifyFreqBoardSeconds` | float64 | `86400` | Delay (seconds) before sending board change notification |
| `webhook_update` | []string | `[]` | List of webhook URLs to notify on block changes |

---

## Logging Settings

| Field | Type | Description |
|-------|------|-------------|
| `logging_cfg_file` | string | Path to Mattermost logging configuration JSON file |
| `logging_cfg_json` | string | Inline logging configuration JSON |

### Example logging JSON:
```json
{
    "EnableConsole": true,
    "ConsoleLevel": "INFO",
    "EnableFile": true,
    "FileLevel": "DEBUG",
    "FileLocation": "./focalboard.log"
}
```

---

## Audit Settings

| Field | Type | Description |
|-------|------|-------------|
| `audit_cfg_file` | string | Path to audit logging configuration JSON file |
| `audit_cfg_json` | string | Inline audit logging configuration JSON |

---

## Telemetry Settings

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `telemetry` | bool | `true` | Enable anonymized usage telemetry |
| `telemetryid` | string | `""` | Unique installation ID (auto-generated if empty) |
| `prometheusaddress` | string | `""` | Address for Prometheus metrics endpoint (e.g., `:9092`) |

---

## Feature Flags

| Field | Type | Description |
|-------|------|-------------|
| `featureFlags` | map[string]string | Key-value feature flags sent to the client |

Example:
```json
{
    "featureFlags": {
        "BoardsManageBoardRoles": "true",
        "BlocksEditor": "true"
    }
}
```

---

## Environment Variable Overrides

Any configuration field can be overridden with an environment variable:
- Prefix: `FOCALBOARD_`
- Field name: uppercase with underscores

Examples:
```bash
FOCALBOARD_PORT=9000
FOCALBOARD_DBTYPE=postgres
FOCALBOARD_DBCONFIG="postgres://user:pass@localhost/fb"
FOCALBOARD_ENABLEPUBLICSHAREDBOARDS=true
FOCALBOARD_FILESDRIVER=amazons3
```

---

## Minimal Configuration (Standalone SQLite)

```json
{
    "serverRoot": "http://localhost:8000",
    "port": 8000,
    "dbtype": "sqlite3",
    "dbconfig": "./focalboard.db",
    "webPath": "./pack",
    "enablePublicSharedBoards": true,
    "telemetry": false
}
```

## Production Configuration (PostgreSQL + S3)

```json
{
    "serverRoot": "https://boards.example.com",
    "port": 8000,
    "useSSL": false,
    "dbtype": "postgres",
    "dbconfig": "postgres://focalboard:secret@db:5432/focalboard?sslmode=disable",
    "filesdriver": "amazons3",
    "filesS3Config": {
        "accessKeyId": "AKIAXXXXXXXX",
        "secretAccessKey": "secret",
        "bucket": "my-focalboard-files",
        "region": "us-east-1",
        "ssl": true
    },
    "enablePublicSharedBoards": true,
    "telemetry": false,
    "session_expire_time": 7776000
}
```
