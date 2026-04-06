# Admin Panel API Reference

## Overview

The admin panel API provides a RESTful interface for managing the SGTP server. It enables
administrators to monitor server activity, manage users and rooms, enforce bans, configure
server settings, and perform administrative operations such as backups and data export.

All endpoints are served under the base path `/api/v1/`.

All request and response bodies use JSON (`Content-Type: application/json`) unless otherwise
noted.

---

## Authentication

The API supports two authentication mechanisms. Every endpoint except `POST /api/v1/auth/login`
and `POST /api/v1/invites/{token}/register` requires authentication.

### Bearer Token

Include an `Authorization` header with a valid access token:

```
Authorization: Bearer <access_token>
```

### Session Cookie

A `session_id` HTTP-only cookie is set upon successful login. The cookie contains the
refresh token and is valid for 7 days. It uses `SameSite=Strict` and `HttpOnly` flags.

### Authentication Priority

The middleware checks Bearer token first. If no Bearer token is present, it falls back to
the `session_id` cookie.

### Unauthenticated Error Response

If authentication fails or is missing, the API returns:

```
HTTP 401 Unauthorized
```

```json
{ "error": "authentication required" }
```

Other possible 401 messages: `"invalid or expired token"`, `"invalid session"`,
`"user not found"`.

---

## Common Patterns

### Pagination

Paginated endpoints accept the following query parameters:

| Parameter  | Type | Default | Max | Description                    |
|------------|------|---------|-----|--------------------------------|
| `page`     | int  | 1       | --  | Page number (1-indexed)        |
| `per_page` | int  | 50      | 500 | Number of items per page       |

Paginated responses use a standard envelope:

```json
{
  "data": [ ... ],
  "meta": {
    "total": 142,
    "page": 1,
    "per_page": 50,
    "total_pages": 3
  }
}
```

### Collection Responses (Non-Paginated)

Some list endpoints that do not paginate return a simpler envelope:

```json
{
  "data": [ ... ],
  "meta": { "total": 5 }
}
```

### Error Responses

All errors are returned as JSON with an `error` field:

```json
{ "error": "description of what went wrong" }
```

Common HTTP status codes:

| Code | Meaning                                        |
|------|------------------------------------------------|
| 400  | Bad request -- invalid input or parameters     |
| 401  | Unauthorized -- missing or invalid credentials |
| 403  | Forbidden -- insufficient permissions          |
| 404  | Not found -- resource does not exist           |
| 500  | Internal server error                          |

### Date Formats

All timestamps in responses are RFC 3339 / ISO 8601 format (e.g., `"2025-01-15T10:30:00Z"`).

Optional timestamp fields in request bodies (such as `expires_at`) also use RFC 3339 format.

Chart date values use the `YYYY-MM-DD` format (e.g., `"2025-01-15"`).

### Success Responses

Many mutating endpoints return a simple success response:

```json
{ "ok": true }
```

---

## Endpoints

### Auth

#### POST /api/v1/auth/login

Log in as an admin user. Does not require authentication.

**Request Body:**

```json
{
  "login_name": "admin",
  "password": "secret"
}
```

**Response (200):**

Returns the login result including tokens. A `session_id` cookie is set automatically
using the refresh token (HttpOnly, SameSite=Strict, path `/`, max-age 7 days).

```json
{
  "access_token": "eyJ...",
  "refresh_token": "abc123...",
  "user": { ... }
}
```

**Error (401):**

```json
{ "error": "invalid credentials" }
```

---

#### POST /api/v1/auth/logout

Log out the current admin session. Requires authentication.

**Request Body (optional):**

```json
{
  "refresh_token": "abc123..."
}
```

If `refresh_token` is not provided in the body, the value from the `session_id` cookie
is used instead.

**Response (200):**

The `session_id` cookie is cleared.

```json
{ "ok": true }
```

---

#### GET /api/v1/auth/me

Get the currently authenticated admin user's profile. Requires authentication.

**Response (200):**

Returns the `AdminUser` object for the authenticated user.

```json
{
  "id": 1,
  "login_name": "admin",
  "display_name": "Admin",
  "role": "admin",
  "force_password_change": false,
  "disabled": false,
  "created_at": "2025-01-01T00:00:00Z"
}
```

---

### Invites

#### GET /api/v1/invites

List all invite links. Requires authentication. Paginated.

**Query Parameters:**

| Parameter  | Type | Default | Description         |
|------------|------|---------|---------------------|
| `page`     | int  | 1       | Page number         |
| `per_page` | int  | 50      | Items per page      |

**Response (200):**

```json
{
  "data": [
    {
      "id": 1,
      "token": "abc123",
      "generated_username": "user_abc",
      "status": "pending",
      "created_at": "2025-01-15T10:00:00Z",
      "expires_at": "2025-01-16T10:00:00Z",
      "used_at": null,
      "revoked_at": null,
      "created_by": 1
    }
  ],
  "meta": {
    "total": 1,
    "page": 1,
    "per_page": 50,
    "total_pages": 1
  }
}
```

The `status` field can be `"pending"`, `"used"`, or `"revoked"`.

---

#### POST /api/v1/invites

Create a new invite link. Requires authentication.

**Request Body:**

```json
{
  "ttl_hours": 24
}
```

| Field      | Type | Description                                       |
|------------|------|---------------------------------------------------|
| `ttl_hours`| int  | Time-to-live in hours before the invite expires    |

**Response (201):**

Returns the created `InviteLink` object.

```json
{
  "id": 2,
  "token": "def456",
  "generated_username": "user_def",
  "status": "pending",
  "created_at": "2025-01-15T12:00:00Z",
  "expires_at": "2025-01-16T12:00:00Z",
  "used_at": null,
  "revoked_at": null,
  "created_by": 1
}
```

---

#### POST /api/v1/invites/{id}/revoke

Revoke an existing invite link. Requires authentication.

**Path Parameters:**

| Parameter | Type  | Description           |
|-----------|-------|-----------------------|
| `id`      | int64 | The invite link ID    |

**Response (200):**

```json
{ "ok": true }
```

**Error (400):**

```json
{ "error": "invalid invite id" }
```

---

#### POST /api/v1/invites/{token}/register

Register a new admin account using an invite token. Does not require authentication.

**Path Parameters:**

| Parameter | Type   | Description          |
|-----------|--------|----------------------|
| `token`   | string | The invite token     |

**Request Body:**

```json
{
  "login_name": "newadmin",
  "password": "securepassword"
}
```

**Response (201):**

Returns the newly created `AdminUser` object.

```json
{
  "id": 2,
  "login_name": "newadmin",
  "display_name": "",
  "role": "admin",
  "force_password_change": false,
  "disabled": false,
  "created_at": "2025-01-15T12:00:00Z"
}
```

---

### Admins

#### GET /api/v1/admins

List all admin users. Requires authentication. Non-paginated.

**Response (200):**

```json
{
  "data": [
    {
      "id": 1,
      "login_name": "admin",
      "display_name": "Admin",
      "role": "admin",
      "force_password_change": false,
      "disabled": false,
      "created_at": "2025-01-01T00:00:00Z"
    }
  ],
  "meta": { "total": 1 }
}
```

---

#### POST /api/v1/admins/{username}/deactivate

Deactivate an admin account. Requires authentication.

**Path Parameters:**

| Parameter  | Type   | Description              |
|------------|--------|--------------------------|
| `username` | string | The admin's login name   |

**Response (200):**

```json
{ "ok": true }
```

**Error (403):**

Returned when the operation is forbidden (e.g., attempting to deactivate yourself or the
last remaining admin):

```json
{ "error": "forbidden: cannot deactivate yourself" }
```

---

### Sessions

#### GET /api/v1/sessions

List all active admin sessions. Requires authentication. Non-paginated.

**Response (200):**

```json
{
  "data": [
    {
      "id": 1,
      "user_id": 1,
      "login_name": "admin",
      "display_name": "Admin",
      "user_agent": "Mozilla/5.0 ...",
      "remote_ip": "192.168.1.1",
      "created_at": "2025-01-15T10:00:00Z",
      "expires_at": "2025-01-22T10:00:00Z"
    }
  ],
  "meta": { "total": 1 }
}
```

---

### Dashboard

#### GET /api/v1/dashboard/metrics

Get aggregate dashboard metrics. Requires authentication.

**Response (200):**

```json
{
  "total_users": 1500,
  "online_users": 42,
  "total_rooms": 350,
  "active_rooms": 28,
  "total_bans": 15,
  "total_requests": 500000,
  "total_bytes_recv": 10485760,
  "total_bytes_sent": 52428800,
  "active_admins": 3,
  "pending_invites": 2
}
```

| Field              | Type  | Description                                             |
|--------------------|-------|---------------------------------------------------------|
| `total_users`      | int64 | Total unique users (by public key or IP)                |
| `online_users`     | int64 | Users seen in the last 5 minutes                        |
| `total_rooms`      | int64 | Total rooms                                             |
| `active_rooms`     | int64 | Rooms with activity in the last hour                    |
| `total_bans`       | int64 | Active user bans (not expired, not unbanned)            |
| `total_requests`   | int64 | Aggregate request count across all users                |
| `total_bytes_recv` | int64 | Aggregate bytes received                                |
| `total_bytes_sent` | int64 | Aggregate bytes sent                                    |
| `active_admins`    | int64 | Number of non-disabled admin accounts                   |
| `pending_invites`  | int64 | Number of pending (unused, non-expired) invite links    |

---

#### GET /api/v1/dashboard/charts/{chart_type}

Get time-series chart data for the dashboard. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                                            |
|--------------|--------|--------------------------------------------------------|
| `chart_type` | string | One of: `requests`, `traffic`, `users`, `rooms`        |

**Query Parameters:**

| Parameter | Type | Default | Max | Description                            |
|-----------|------|---------|-----|----------------------------------------|
| `days`    | int  | 30      | 365 | Number of days of history to return    |

**Chart Types:**

| Type       | Description                                    |
|------------|------------------------------------------------|
| `requests` | Daily request counts (global scope)            |
| `traffic`  | Daily total traffic in bytes (recv + sent)     |
| `users`    | Daily new user registrations                   |
| `rooms`    | Daily new room creations                       |

**Response (200):**

```json
{
  "chart_type": "requests",
  "days": 30,
  "data": [
    { "date": "2025-01-01", "value": 1234 },
    { "date": "2025-01-02", "value": 5678 }
  ]
}
```

**Error (400):**

```json
{ "error": "unknown chart type: foo" }
```

---

### Users

#### GET /api/v1/users

List users with search, filtering, and sorting. Requires authentication. Paginated.

**Query Parameters:**

| Parameter    | Type   | Default | Description                                           |
|--------------|--------|---------|-------------------------------------------------------|
| `page`       | int    | 1       | Page number                                           |
| `per_page`   | int    | 50      | Items per page (max 500)                              |
| `search`     | string | --      | Search by user key or IP (ILIKE match)                |
| `ip`         | string | --      | Filter by IP address (ILIKE match)                    |
| `public_key` | string | --      | Filter by public key (ILIKE match)                    |
| `sort`       | string | --      | Sort field                                            |
| `order`      | string | --      | Sort order: `asc` or `desc`                           |

**Response (200):**

Standard paginated response with user objects in `data`.

---

#### GET /api/v1/users/{public_key}

Get detailed information for a specific user. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |

**Response (200):**

Returns user details enriched with ban status and per-user limits:

```json
{
  "public_key": "abc123...",
  "ips": ["192.168.1.1", "10.0.0.1"],
  "first_use": "2025-01-01T00:00:00Z",
  "last_use": "2025-01-15T10:00:00Z",
  "requests": 5000,
  "bytes_recv": 1048576,
  "bytes_sent": 2097152,
  "is_banned": false,
  "limits": {
    "traffic_limit_bytes": 0,
    "traffic_limit_period": "",
    "request_rate_limit": 0
  }
}
```

**Error (404):**

```json
{ "error": "user not found" }
```

---

#### GET /api/v1/users/{public_key}/ips

List all IP addresses used by a specific user. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |

**Response (200):**

```json
{
  "data": [
    {
      "ip": "192.168.1.1",
      "first_use": "2025-01-01T00:00:00Z",
      "last_use": "2025-01-15T10:00:00Z",
      "requests": 3000,
      "bytes_recv": 524288,
      "bytes_sent": 1048576
    }
  ]
}
```

---

#### GET /api/v1/users/{public_key}/rooms

List all rooms a user has participated in. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |

**Response (200):**

```json
{
  "data": [
    {
      "room_id": "550e8400-e29b-41d4-a716-446655440000",
      "first_use": "2025-01-01T00:00:00Z",
      "last_use": "2025-01-15T10:00:00Z",
      "requests": 1200,
      "bytes_recv": 262144,
      "bytes_sent": 524288
    }
  ]
}
```

---

#### GET /api/v1/users/{public_key}/chain

Perform chain detection (BFS) to find users sharing IP addresses. Requires authentication.
Trusted IPs are automatically excluded from the graph traversal.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The starting user's public key |

**Query Parameters:**

| Parameter | Type | Default | Description                                    |
|-----------|------|---------|------------------------------------------------|
| `depth`   | int  | 5       | Maximum BFS traversal depth                    |

**Response (200):**

```json
{
  "public_key": "abc123...",
  "depth": 5,
  "edges": [
    {
      "from": "abc123...",
      "to": "def456...",
      "shared_ip": "192.168.1.1",
      "depth": 1
    },
    {
      "from": "def456...",
      "to": "ghi789...",
      "shared_ip": "10.0.0.1",
      "depth": 2
    }
  ]
}
```

If no connections are found, `edges` is an empty array `[]`.

---

#### GET /api/v1/users/{public_key}/bans

List a user's ban history (all bans, including expired and unbanned). Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |

**Response (200):**

```json
{
  "data": [
    {
      "id": 1,
      "public_key": "abc123...",
      "reason": "Spam",
      "banned_at": "2025-01-10T10:00:00Z",
      "expires_at": "2025-02-10T10:00:00Z",
      "unbanned_at": "2025-01-12T10:00:00Z",
      "unbanned_reason": "Appeal accepted",
      "banned_by": "admin",
      "unbanned_by": "admin"
    }
  ]
}
```

---

#### POST /api/v1/users/{public_key}/ban

Ban a user. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |

**Request Body:**

```json
{
  "reason": "Spam activity",
  "expires_at": "2025-02-15T00:00:00Z"
}
```

| Field       | Type     | Required | Description                                       |
|-------------|----------|----------|---------------------------------------------------|
| `reason`    | string   | No       | Reason for the ban                                |
| `expires_at`| datetime | No       | Expiration time (RFC 3339). Omit for permanent ban|

**Response (201):**

Returns the created `UserBan` object.

```json
{
  "id": 2,
  "public_key": "abc123...",
  "reason": "Spam activity",
  "banned_at": "2025-01-15T12:00:00Z",
  "expires_at": "2025-02-15T00:00:00Z",
  "unbanned_at": null,
  "unbanned_reason": "",
  "banned_by": "admin",
  "unbanned_by": ""
}
```

---

#### POST /api/v1/users/{public_key}/unban

Unban a user. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |

**Request Body:**

```json
{
  "reason": "Appeal accepted"
}
```

| Field    | Type   | Required | Description              |
|----------|--------|----------|--------------------------|
| `reason` | string | No       | Reason for the unban     |

**Response (200):**

```json
{ "ok": true }
```

---

#### GET /api/v1/users/{public_key}/room-bans

List all room-specific bans for a user. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |

**Response (200):**

```json
{
  "data": [
    {
      "id": 1,
      "public_key": "abc123...",
      "room_uuid": "550e8400-e29b-41d4-a716-446655440000",
      "reason": "Off-topic",
      "banned_at": "2025-01-10T10:00:00Z",
      "expires_at": null,
      "unbanned_at": null,
      "unbanned_reason": "",
      "banned_by": "admin",
      "unbanned_by": ""
    }
  ]
}
```

---

#### POST /api/v1/users/{public_key}/rooms/{room_uuid}/ban

Ban a user from a specific room. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |
| `room_uuid`  | string | The room UUID              |

**Request Body:**

```json
{
  "reason": "Disruptive behavior",
  "expires_at": "2025-02-15T00:00:00Z"
}
```

| Field       | Type     | Required | Description                                       |
|-------------|----------|----------|---------------------------------------------------|
| `reason`    | string   | No       | Reason for the ban                                |
| `expires_at`| datetime | No       | Expiration time (RFC 3339). Omit for permanent ban|

**Response (201):**

Returns the created `UserRoomBan` object.

---

#### POST /api/v1/users/{public_key}/rooms/{room_uuid}/unban

Unban a user from a specific room. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |
| `room_uuid`  | string | The room UUID              |

**Request Body:**

```json
{
  "reason": "Ban lifted"
}
```

**Response (200):**

```json
{ "ok": true }
```

---

#### PUT /api/v1/users/{public_key}/limits

Set per-user traffic and rate limits. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |

**Request Body:**

```json
{
  "traffic_limit_bytes": 104857600,
  "traffic_limit_period": "day",
  "request_rate_limit": 1000
}
```

| Field                  | Type   | Description                                       |
|------------------------|--------|---------------------------------------------------|
| `traffic_limit_bytes`  | int64  | Maximum traffic in bytes for the period           |
| `traffic_limit_period` | string | Period for the traffic limit (e.g., `"day"`)      |
| `request_rate_limit`   | int    | Maximum number of requests allowed per period     |

**Response (200):**

```json
{ "ok": true }
```

---

#### DELETE /api/v1/users/{public_key}/limits

Remove per-user traffic and rate limits (reset to defaults). Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |

**Response (200):**

```json
{ "ok": true }
```

---

#### GET /api/v1/users/{public_key}/audit-log

List audit log entries related to a specific user. Requires authentication. Paginated.

**Path Parameters:**

| Parameter    | Type   | Description                |
|--------------|--------|----------------------------|
| `public_key` | string | The user's public key      |

**Query Parameters:**

| Parameter  | Type   | Default | Description                         |
|------------|--------|---------|-------------------------------------|
| `page`     | int    | 1       | Page number                         |
| `per_page` | int    | 50      | Items per page (max 500)            |
| `action`   | string | --      | Filter by action type (ILIKE match) |

**Response (200):**

Standard paginated response containing audit log entries. See the [Audit Log](#audit-log)
section for the entry structure.

---

### Bulk User Operations

#### POST /api/v1/users/bulk/ban

Ban multiple users at once. Requires authentication.

**Request Body:**

```json
{
  "public_keys": ["abc123...", "def456..."],
  "reason": "Coordinated spam",
  "expires_at": "2025-02-15T00:00:00Z"
}
```

| Field         | Type     | Required | Description                                      |
|---------------|----------|----------|--------------------------------------------------|
| `public_keys` | string[] | Yes      | Array of public keys to ban                      |
| `reason`      | string   | No       | Reason for the bans                              |
| `expires_at`  | datetime | No       | Expiration time (RFC 3339). Omit for permanent   |

**Response (200):**

```json
{ "ok": true, "count": 2 }
```

---

#### POST /api/v1/users/bulk/unban

Unban multiple users at once. Requires authentication.

**Request Body:**

```json
{
  "public_keys": ["abc123...", "def456..."],
  "reason": "Mass unban"
}
```

| Field         | Type     | Required | Description                      |
|---------------|----------|----------|----------------------------------|
| `public_keys` | string[] | Yes      | Array of public keys to unban    |
| `reason`      | string   | No       | Reason for the unbans            |

**Response (200):**

```json
{ "ok": true, "count": 2 }
```

---

#### PUT /api/v1/users/bulk/limits

Set traffic and rate limits for multiple users at once. Requires authentication.

**Request Body:**

```json
{
  "public_keys": ["abc123...", "def456..."],
  "traffic_limit_bytes": 104857600,
  "traffic_limit_period": "day",
  "request_rate_limit": 1000
}
```

| Field                  | Type     | Required | Description                                  |
|------------------------|----------|----------|----------------------------------------------|
| `public_keys`          | string[] | Yes      | Array of public keys                         |
| `traffic_limit_bytes`  | int64    | Yes      | Maximum traffic in bytes for the period      |
| `traffic_limit_period` | string   | Yes      | Period for the traffic limit (e.g., `"day"`) |
| `request_rate_limit`   | int      | Yes      | Maximum requests allowed per period          |

**Response (200):**

```json
{ "ok": true, "count": 2 }
```

---

### Bans

#### GET /api/v1/bans/users

List all active user bans. Requires authentication. Paginated.

**Query Parameters:**

| Parameter  | Type | Default | Description         |
|------------|------|---------|---------------------|
| `page`     | int  | 1       | Page number         |
| `per_page` | int  | 50      | Items per page      |

**Response (200):**

Standard paginated response with `UserBan` objects:

```json
{
  "data": [
    {
      "id": 1,
      "public_key": "abc123...",
      "reason": "Spam",
      "banned_at": "2025-01-10T10:00:00Z",
      "expires_at": null,
      "unbanned_at": null,
      "unbanned_reason": "",
      "banned_by": "admin",
      "unbanned_by": ""
    }
  ],
  "meta": { "total": 1, "page": 1, "per_page": 50, "total_pages": 1 }
}
```

---

#### GET /api/v1/bans/ips

List all active IP bans. Requires authentication. Paginated.

**Query Parameters:**

| Parameter  | Type | Default | Description         |
|------------|------|---------|---------------------|
| `page`     | int  | 1       | Page number         |
| `per_page` | int  | 50      | Items per page      |

**Response (200):**

Standard paginated response with `IPBan` objects:

```json
{
  "data": [
    {
      "id": 1,
      "ip_address": "10.0.0.5",
      "reason": "Abuse",
      "banned_at": "2025-01-10T10:00:00Z",
      "expires_at": "2025-02-10T10:00:00Z",
      "unbanned_at": null,
      "unbanned_reason": "",
      "banned_by": "admin",
      "unbanned_by": ""
    }
  ],
  "meta": { "total": 1, "page": 1, "per_page": 50, "total_pages": 1 }
}
```

---

#### POST /api/v1/bans/ips

Ban an IP address. Requires authentication.

**Request Body:**

```json
{
  "ip_address": "10.0.0.5",
  "reason": "Abuse from this IP",
  "expires_at": "2025-02-15T00:00:00Z"
}
```

| Field       | Type     | Required | Description                                       |
|-------------|----------|----------|---------------------------------------------------|
| `ip_address`| string   | Yes      | The IP address to ban                             |
| `reason`    | string   | No       | Reason for the ban                                |
| `expires_at`| datetime | No       | Expiration time (RFC 3339). Omit for permanent ban|

**Response (201):**

Returns the created `IPBan` object.

```json
{
  "id": 2,
  "ip_address": "10.0.0.5",
  "reason": "Abuse from this IP",
  "banned_at": "2025-01-15T12:00:00Z",
  "expires_at": "2025-02-15T00:00:00Z",
  "unbanned_at": null,
  "unbanned_reason": "",
  "banned_by": "admin",
  "unbanned_by": ""
}
```

---

#### GET /api/v1/bans/ips/{ip_address}/history

Get the complete ban history for an IP address. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                  |
|--------------|--------|------------------------------|
| `ip_address` | string | The IP address to look up    |

**Response (200):**

```json
{
  "data": [
    {
      "id": 1,
      "ip_address": "10.0.0.5",
      "reason": "First ban",
      "banned_at": "2025-01-01T10:00:00Z",
      "expires_at": null,
      "unbanned_at": "2025-01-05T10:00:00Z",
      "unbanned_reason": "Lifted",
      "banned_by": "admin",
      "unbanned_by": "admin"
    },
    {
      "id": 2,
      "ip_address": "10.0.0.5",
      "reason": "Second ban",
      "banned_at": "2025-01-10T10:00:00Z",
      "expires_at": null,
      "unbanned_at": null,
      "unbanned_reason": "",
      "banned_by": "admin",
      "unbanned_by": ""
    }
  ]
}
```

---

#### POST /api/v1/bans/ips/{ip_address}/unban

Unban an IP address. Requires authentication.

**Path Parameters:**

| Parameter    | Type   | Description                  |
|--------------|--------|------------------------------|
| `ip_address` | string | The IP address to unban      |

**Request Body:**

```json
{
  "reason": "Ban lifted"
}
```

| Field    | Type   | Required | Description              |
|----------|--------|----------|--------------------------|
| `reason` | string | No       | Reason for the unban     |

**Response (200):**

```json
{ "ok": true }
```

---

#### GET /api/v1/bans/rooms

List all active room bans. Requires authentication. Paginated.

**Query Parameters:**

| Parameter  | Type | Default | Description         |
|------------|------|---------|---------------------|
| `page`     | int  | 1       | Page number         |
| `per_page` | int  | 50      | Items per page      |

**Response (200):**

Standard paginated response with `RoomBan` objects:

```json
{
  "data": [
    {
      "id": 1,
      "room_uuid": "550e8400-e29b-41d4-a716-446655440000",
      "reason": "Illegal content",
      "banned_at": "2025-01-10T10:00:00Z",
      "expires_at": null,
      "unbanned_at": null,
      "unbanned_reason": "",
      "banned_by": "admin",
      "unbanned_by": ""
    }
  ],
  "meta": { "total": 1, "page": 1, "per_page": 50, "total_pages": 1 }
}
```

---

### Rooms

#### GET /api/v1/rooms

List rooms with search and sorting. Requires authentication. Paginated.

**Query Parameters:**

| Parameter  | Type   | Default | Description                                   |
|------------|--------|---------|-----------------------------------------------|
| `page`     | int    | 1       | Page number                                   |
| `per_page` | int    | 50      | Items per page (max 500)                      |
| `search`   | string | --      | Search by room ID (ILIKE match)               |
| `sort`     | string | --      | Sort field                                    |
| `order`    | string | --      | Sort order: `asc` or `desc`                   |

**Response (200):**

Standard paginated response with room objects in `data`.

---

#### GET /api/v1/rooms/{uuid}

Get detailed information for a specific room. Requires authentication.

**Path Parameters:**

| Parameter | Type   | Description      |
|-----------|--------|------------------|
| `uuid`    | string | The room UUID    |

**Response (200):**

Returns room details object.

**Error (404):**

```json
{ "error": "room not found" }
```

---

#### GET /api/v1/rooms/{uuid}/participants

List participants in a room. Requires authentication. Paginated.

**Path Parameters:**

| Parameter | Type   | Description      |
|-----------|--------|------------------|
| `uuid`    | string | The room UUID    |

**Query Parameters:**

| Parameter  | Type | Default | Description         |
|------------|------|---------|---------------------|
| `page`     | int  | 1       | Page number         |
| `per_page` | int  | 50      | Items per page      |

**Response (200):**

```json
{
  "data": [ ... ]
}
```

---

#### GET /api/v1/rooms/{uuid}/participants/online

List currently online participants in a room (seen within the last 60 seconds).
Requires authentication.

**Path Parameters:**

| Parameter | Type   | Description      |
|-----------|--------|------------------|
| `uuid`    | string | The room UUID    |

**Response (200):**

```json
{
  "data": [
    {
      "ip": "192.168.1.1",
      "public_key": "abc123...",
      "last_use": "2025-01-15T12:00:00Z",
      "requests": 50,
      "bytes_recv": 4096,
      "bytes_sent": 8192
    }
  ]
}
```

---

#### GET /api/v1/rooms/{uuid}/ips

List all distinct IP addresses that have accessed a room. Requires authentication.

**Path Parameters:**

| Parameter | Type   | Description      |
|-----------|--------|------------------|
| `uuid`    | string | The room UUID    |

**Response (200):**

```json
{
  "data": [
    {
      "ip": "192.168.1.1",
      "first_use": "2025-01-01T00:00:00Z",
      "last_use": "2025-01-15T10:00:00Z",
      "requests": 3000,
      "bytes_recv": 524288,
      "bytes_sent": 1048576
    }
  ]
}
```

---

#### GET /api/v1/rooms/{uuid}/bans

List all bans (including historical) for a specific room. Requires authentication.

**Path Parameters:**

| Parameter | Type   | Description      |
|-----------|--------|------------------|
| `uuid`    | string | The room UUID    |

**Response (200):**

```json
{
  "data": [ ... ]
}
```

Returns `RoomBan` objects for this specific room.

---

#### POST /api/v1/rooms/{uuid}/ban

Ban a room. Requires authentication.

**Path Parameters:**

| Parameter | Type   | Description      |
|-----------|--------|------------------|
| `uuid`    | string | The room UUID    |

**Request Body:**

```json
{
  "reason": "Illegal content",
  "expires_at": "2025-02-15T00:00:00Z"
}
```

| Field       | Type     | Required | Description                                       |
|-------------|----------|----------|---------------------------------------------------|
| `reason`    | string   | No       | Reason for the ban                                |
| `expires_at`| datetime | No       | Expiration time (RFC 3339). Omit for permanent ban|

**Response (201):**

Returns the created `RoomBan` object.

---

#### POST /api/v1/rooms/{uuid}/unban

Unban a room. Requires authentication.

**Path Parameters:**

| Parameter | Type   | Description      |
|-----------|--------|------------------|
| `uuid`    | string | The room UUID    |

**Request Body:**

```json
{
  "reason": "Ban lifted"
}
```

| Field    | Type   | Required | Description              |
|----------|--------|----------|--------------------------|
| `reason` | string | No       | Reason for the unban     |

**Response (200):**

```json
{ "ok": true }
```

---

#### POST /api/v1/rooms/{uuid}/ban-all-users

Ban all users who have participated in a specific room. Requires authentication.

This finds all distinct public keys with activity in the room and creates user bans
for each of them.

**Path Parameters:**

| Parameter | Type   | Description      |
|-----------|--------|------------------|
| `uuid`    | string | The room UUID    |

**Request Body:**

```json
{
  "reason": "All participants in banned room",
  "expires_at": "2025-02-15T00:00:00Z"
}
```

| Field       | Type     | Required | Description                                       |
|-------------|----------|----------|---------------------------------------------------|
| `reason`    | string   | No       | Reason for the bans                               |
| `expires_at`| datetime | No       | Expiration time (RFC 3339). Omit for permanent ban|

**Response (200):**

```json
{ "ok": true, "banned_count": 15 }
```

---

#### GET /api/v1/rooms/{uuid}/audit-log

List audit log entries related to a specific room. Requires authentication. Paginated.

**Path Parameters:**

| Parameter | Type   | Description      |
|-----------|--------|------------------|
| `uuid`    | string | The room UUID    |

**Query Parameters:**

| Parameter  | Type   | Default | Description                         |
|------------|--------|---------|-------------------------------------|
| `page`     | int    | 1       | Page number                         |
| `per_page` | int    | 50      | Items per page (max 500)            |
| `action`   | string | --      | Filter by action type (ILIKE match) |

**Response (200):**

Standard paginated response containing audit log entries. See the [Audit Log](#audit-log)
section for the entry structure.

---

### Audit Log

#### GET /api/v1/audit-log

List audit log entries with optional filters. Requires authentication. Paginated.

**Query Parameters:**

| Parameter     | Type   | Default | Description                              |
|---------------|--------|---------|------------------------------------------|
| `page`        | int    | 1       | Page number                              |
| `per_page`    | int    | 50      | Items per page (max 500)                 |
| `action`      | string | --      | Filter by action type (ILIKE match)      |
| `object_type` | string | --      | Filter by object type (exact match)      |
| `object_id`   | string | --      | Filter by object ID (exact match)        |

**Response (200):**

```json
{
  "data": [
    {
      "id": 1,
      "actor_user_id": 1,
      "actor_login": "admin",
      "actor_name": "Admin",
      "action": "ban_user",
      "object_type": "user",
      "object_id": "abc123...",
      "payload": { "reason": "Spam" },
      "created_at": "2025-01-15T10:00:00Z"
    }
  ],
  "meta": {
    "total": 1,
    "page": 1,
    "per_page": 50,
    "total_pages": 1
  }
}
```

**Audit Log Entry Fields:**

| Field            | Type   | Description                                         |
|------------------|--------|-----------------------------------------------------|
| `id`             | int64  | Unique entry ID                                     |
| `actor_user_id`  | int64  | ID of the admin who performed the action            |
| `actor_login`    | string | Login name of the actor                             |
| `actor_name`     | string | Display name of the actor                           |
| `action`         | string | The action performed (e.g., `ban_user`, `unban_ip`) |
| `object_type`    | string | Type of the affected object (e.g., `user`, `room`)  |
| `object_id`      | string | ID of the affected object                           |
| `payload`        | object | Additional context about the action (JSON)          |
| `created_at`     | string | Timestamp of when the action occurred               |

---

### Settings

#### GET /api/v1/settings

Get all admin panel settings. Requires authentication.

**Response (200):**

Returns a key-value map of all settings. Values are JSON values of varying types:

```json
{
  "server_name": "\"My SGTP Server\"",
  "maintenance_mode": "false",
  "max_room_size": "100"
}
```

Each value is a raw JSON value (string, number, boolean, or object).

---

#### PATCH /api/v1/settings

Update one or more admin panel settings. Requires authentication.

**Request Body:**

A JSON object where keys are setting names and values are the new JSON values:

```json
{
  "server_name": "\"Updated Server Name\"",
  "maintenance_mode": "true"
}
```

Values are raw JSON -- strings must be JSON-encoded strings (i.e., wrapped in quotes),
numbers and booleans are bare values.

**Response (200):**

```json
{ "ok": true }
```

---

#### GET /api/v1/settings/trusted-ips

List all trusted IP addresses. Requires authentication.

Trusted IPs are excluded from chain detection (BFS shared-IP analysis).

**Response (200):**

```json
{
  "data": [
    {
      "ip_address": "10.0.0.1",
      "added_at": "2025-01-01T00:00:00Z",
      "added_by": "admin"
    }
  ]
}
```

---

#### POST /api/v1/settings/trusted-ips

Add a trusted IP address. Requires authentication.

If the IP already exists, it is updated with the new `added_by` value.

**Request Body:**

```json
{
  "ip_address": "10.0.0.1"
}
```

| Field       | Type   | Required | Description                    |
|-------------|--------|----------|--------------------------------|
| `ip_address`| string | Yes      | The IP address to trust        |

**Response (201):**

Returns the created (or updated) `TrustedIP` object:

```json
{
  "ip_address": "10.0.0.1",
  "added_at": "2025-01-15T12:00:00Z",
  "added_by": "admin"
}
```

---

#### DELETE /api/v1/settings/trusted-ips/{ip}

Remove a trusted IP address. Requires authentication.

**Path Parameters:**

| Parameter | Type   | Description                     |
|-----------|--------|---------------------------------|
| `ip`      | string | The IP address to remove        |

**Response (200):**

```json
{ "ok": true }
```

---

### Backups

#### GET /api/v1/backups

List backup jobs. Requires authentication.

**Query Parameters:**

| Parameter | Type | Default | Description                              |
|-----------|------|---------|------------------------------------------|
| `limit`   | int  | 50      | Maximum number of backup jobs to return  |

**Response (200):**

```json
{
  "data": [ ... ]
}
```

Returns an array of backup job objects.

---

#### POST /api/v1/backups/now

Trigger an immediate backup. Requires authentication.

**Request Body:** None required.

**Response (202 Accepted):**

```json
{
  "job_id": 5,
  "status": "queued"
}
```

---

#### DELETE /api/v1/backups/{id}

Delete a backup job record. Requires authentication.

**Path Parameters:**

| Parameter | Type  | Description          |
|-----------|-------|----------------------|
| `id`      | int64 | The backup job ID    |

**Response (200):**

```json
{ "ok": true }
```

**Error (400):**

```json
{ "error": "invalid backup id" }
```

---

#### GET /api/v1/backups/settings

Get the current backup configuration. Requires authentication.

**Response (200):**

```json
{
  "id": 1,
  "cron_expression": "0 2 * * *",
  "retention_count": 10,
  "retention_max_age_days": 30,
  "local_path": "/var/backups/sgtp",
  "s3_endpoint": null,
  "s3_bucket": null,
  "s3_access_key": null,
  "s3_secret_key": null,
  "s3_prefix": null,
  "s3_region": null,
  "updated_at": "2025-01-15T10:00:00Z"
}
```

**BackupSettings Fields:**

| Field                   | Type    | Nullable | Description                                    |
|-------------------------|---------|----------|------------------------------------------------|
| `id`                    | int     | No       | Settings row ID (always 1)                     |
| `cron_expression`       | string  | No       | Cron schedule expression for automated backups |
| `retention_count`       | int     | Yes      | Maximum number of backups to retain            |
| `retention_max_age_days`| int     | Yes      | Maximum age in days for backup retention       |
| `local_path`            | string  | No       | Local filesystem path for backup storage       |
| `s3_endpoint`           | string  | Yes      | S3-compatible endpoint URL                     |
| `s3_bucket`             | string  | Yes      | S3 bucket name                                 |
| `s3_access_key`         | string  | Yes      | S3 access key                                  |
| `s3_secret_key`         | string  | Yes      | S3 secret key                                  |
| `s3_prefix`             | string  | Yes      | S3 object key prefix                           |
| `s3_region`             | string  | Yes      | S3 region                                      |
| `updated_at`            | string  | No       | Last update timestamp                          |

---

#### PUT /api/v1/backups/settings

Update the backup configuration. Requires authentication.

**Request Body:**

```json
{
  "cron_expression": "0 3 * * *",
  "retention_count": 14,
  "retention_max_age_days": 60,
  "local_path": "/var/backups/sgtp",
  "s3_endpoint": "https://s3.example.com",
  "s3_bucket": "my-backups",
  "s3_access_key": "AKIAIOSFODNN7EXAMPLE",
  "s3_secret_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  "s3_prefix": "sgtp/",
  "s3_region": "us-east-1"
}
```

All fields from `BackupSettings` are accepted. Nullable fields can be set to `null` to
clear them.

**Response (200):**

```json
{ "ok": true }
```

---

### Export

#### POST /api/v1/export

Trigger a data export (XLSX download). Requires authentication.

**Request Body:** None required.

**Response (202 Accepted):**

```json
{
  "job_id": 3,
  "status": "queued"
}
```

The export job is queued and processed asynchronously. The `job_id` can be used to track
the export status.

---

## Data Types Reference

### UserBan

```json
{
  "id": 1,
  "public_key": "string",
  "reason": "string",
  "banned_at": "2025-01-15T10:00:00Z",
  "expires_at": "2025-02-15T10:00:00Z",
  "unbanned_at": "2025-01-20T10:00:00Z",
  "unbanned_reason": "string",
  "banned_by": "string",
  "unbanned_by": "string"
}
```

### IPBan

```json
{
  "id": 1,
  "ip_address": "string",
  "reason": "string",
  "banned_at": "2025-01-15T10:00:00Z",
  "expires_at": "2025-02-15T10:00:00Z",
  "unbanned_at": "2025-01-20T10:00:00Z",
  "unbanned_reason": "string",
  "banned_by": "string",
  "unbanned_by": "string"
}
```

### RoomBan

```json
{
  "id": 1,
  "room_uuid": "string",
  "reason": "string",
  "banned_at": "2025-01-15T10:00:00Z",
  "expires_at": "2025-02-15T10:00:00Z",
  "unbanned_at": "2025-01-20T10:00:00Z",
  "unbanned_reason": "string",
  "banned_by": "string",
  "unbanned_by": "string"
}
```

### UserRoomBan

```json
{
  "id": 1,
  "public_key": "string",
  "room_uuid": "string",
  "reason": "string",
  "banned_at": "2025-01-15T10:00:00Z",
  "expires_at": "2025-02-15T10:00:00Z",
  "unbanned_at": "2025-01-20T10:00:00Z",
  "unbanned_reason": "string",
  "banned_by": "string",
  "unbanned_by": "string"
}
```

### InviteLink

```json
{
  "id": 1,
  "token": "string",
  "generated_username": "string",
  "status": "pending",
  "created_at": "2025-01-15T10:00:00Z",
  "expires_at": "2025-01-16T10:00:00Z",
  "used_at": null,
  "revoked_at": null,
  "created_by": 1
}
```

Status values: `"pending"`, `"used"`, `"revoked"`.

### TrustedIP

```json
{
  "ip_address": "string",
  "added_at": "2025-01-15T10:00:00Z",
  "added_by": "string"
}
```

### DashboardMetrics

```json
{
  "total_users": 0,
  "online_users": 0,
  "total_rooms": 0,
  "active_rooms": 0,
  "total_bans": 0,
  "total_requests": 0,
  "total_bytes_recv": 0,
  "total_bytes_sent": 0,
  "active_admins": 0,
  "pending_invites": 0
}
```

### BackupSettings

```json
{
  "id": 1,
  "cron_expression": "string",
  "retention_count": null,
  "retention_max_age_days": null,
  "local_path": "string",
  "s3_endpoint": null,
  "s3_bucket": null,
  "s3_access_key": null,
  "s3_secret_key": null,
  "s3_prefix": null,
  "s3_region": null,
  "updated_at": "2025-01-15T10:00:00Z"
}
```
