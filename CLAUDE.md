# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WuzAPI is a Go-based RESTful API wrapper around the [whatsmeow](https://go.mau.fi/whatsmeow) WhatsApp WebSocket library. It provides multi-device WhatsApp support with concurrent session management, webhook event delivery, and optional S3/RabbitMQ integrations.

## Build & Run

```bash
# Build
go build .

# Run (SQLite default, auto-generates credentials)
./wuzapi

# Run with flags
./wuzapi -port 8080 -address 0.0.0.0 -admintoken <token> -logtype console -color

# Run in JSON-RPC stdio mode (for subprocess integration)
./wuzapi -mode stdio

# Docker
docker compose up
```

## Testing

**Always run `go build .` and `go test ./...` after any code change.** Ensure compilation and all tests pass before considering work complete.

```bash
# Run all tests
go test ./...

# Run a specific test
go test -run TestFunctionName ./...
```

Tests are minimal — only `stdio_test.go` exists currently.

## Architecture

**Single-package Go application** — all code lives in `package main` at the repository root. No internal packages.

### Key Files

- **main.go** — Entry point, CLI flag parsing, server initialization, graceful shutdown. Supports two modes: HTTP server and JSON-RPC 2.0 over stdio.
- **routes.go** — All HTTP route definitions using gorilla/mux. Two middleware chains: `authalice` (user token auth) and `authadmin` (admin token auth).
- **handlers.go** (largest file, ~194KB) — All HTTP endpoint handler implementations. Every API endpoint method lives here.
- **wmiau.go** (~60KB) — WhatsApp client lifecycle management (`MyClient` struct), event handler registration, and event-to-webhook routing.
- **helpers.go** — Utility functions: webhook delivery with retries, AES-256 encryption/decryption, HMAC signing, Open Graph fetching.
- **migrations.go** — Sequential database schema migrations (numbered 1-8).
- **db.go** — Database initialization, auto-selects SQLite or PostgreSQL based on `DB_*` environment variables.
- **clients.go** — `ClientManager` struct: thread-safe map managing concurrent WhatsApp sessions (`whatsmeowClients`, `httpClients`, `myClients`).
- **stdio.go** — JSON-RPC 2.0 interface that bridges stdin/stdout to HTTP handlers.
- **s3manager.go** — Per-user S3 media upload configuration and operations.
- **rabbitmq.go** — RabbitMQ connection management and event publishing.
- **constants.go** — Event type string constants (Message, Connected, GroupInfo, etc.).

### Request Flow

1. HTTP request → gorilla/mux router (routes.go)
2. Authentication middleware validates token, sets user ID in context
3. Handler function in handlers.go processes request
4. Handler interacts with WhatsApp via `ClientManager` → `MyClient` → `whatsmeow.Client`
5. WhatsApp events flow back through `MyClient.myEventHandler()` in wmiau.go
6. Events dispatched to: user webhook, global webhook, and/or RabbitMQ

### Database

- **SQLite** (default): stores in `dbdata/users.db` relative to executable or `-datadir`
- **PostgreSQL**: enabled when all `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_HOST`, `DB_PORT` env vars are set
- Two logical databases: the app DB (users, message_history) and whatsmeow's own SQLite store for session data

### Authentication

- **User endpoints**: `Authorization: <user_token>` header, token stored in users table
- **Admin endpoints** (`/admin/**`): `Authorization: <admin_token>` header, set via `WUZAPI_ADMIN_TOKEN` env var or auto-generated at startup

### Event System

Events from WhatsApp are delivered through three parallel channels:
1. **Per-user webhook** — HTTP POST to user's configured URL, filtered by subscription list
2. **Global webhook** — `WUZAPI_GLOBAL_WEBHOOK` env var, receives all events from all users
3. **RabbitMQ** — Optional queue-based delivery via `RABBITMQ_URL`

Webhooks support HMAC-SHA256 signing (per-user or global key) and configurable retry logic.

## Environment Variables

Key variables (see `.env.sample` for full list):
- `WUZAPI_ADMIN_TOKEN` — Admin API token (auto-generated if unset)
- `WUZAPI_GLOBAL_ENCRYPTION_KEY` — AES-256 key for encrypting sensitive DB fields
- `WUZAPI_GLOBAL_WEBHOOK` — URL receiving all events from all users
- `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_HOST`, `DB_PORT` — PostgreSQL config (all required to use PG)
- `RABBITMQ_URL`, `RABBITMQ_QUEUE` — RabbitMQ connection
- `WEBHOOK_FORMAT` — `json` or `form`
- `WEBHOOK_RETRY_ENABLED`, `WEBHOOK_RETRY_COUNT`, `WEBHOOK_RETRY_DELAY_SECONDS` — Retry config
