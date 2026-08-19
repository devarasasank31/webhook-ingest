# webhook-ingest

A Go service that receives call-completion webhooks from a telephony provider, stores them in PostgreSQL, maintains per-account statistics, and processes recordings in the background.

## What this project solves

Webhook providers use at-least-once delivery, so the same event may be delivered multiple times. This service ensures a repeated `event_id` does not create duplicate side effects.

The implementation fixes:

- duplicate webhook event processing;
- double-counted account statistics;
- recording processing failures that were previously silent;
- in-flight recording work being lost during shutdown.

## Idempotency approach

PostgreSQL is the source of truth for webhook idempotency.

- `events.event_id` has a unique database constraint.
- Ingestion uses `INSERT ... ON CONFLICT DO NOTHING`.
- The event insert, call upsert, and durable statistics update run in one transaction.
- Only the request that successfully creates the event performs side effects.
- Duplicate deliveries return success without incrementing statistics again.

This prevents races when identical webhook deliveries arrive concurrently.

## Architecture

```text
POST /webhooks/calls
  -> HTTP handler validates the request
  -> ingestion service
  -> PostgreSQL transaction
       -> events
       -> calls
       -> account_stats
  -> in-memory statistics cache
  -> background recording processing
```

Redis remains connected as part of the existing service architecture, but it is not used as the idempotency source because PostgreSQL can atomically store the event and update its related durable data.

## API

### Health check

```http
GET /healthz
```

Response:

```text
ok
```

### Ingest a call webhook

```http
POST /webhooks/calls
Content-Type: application/json
```

Example body:

```json
{
  "event_id": "evt_01H8XK2M9P",
  "call_id": "call_9f2ab31c",
  "account_id": "acc_123",
  "status": "completed",
  "duration_sec": 143,
  "recording_url": "https://recordings.example.com/9f2ab31c.wav",
  "occurred_at": "2026-08-13T09:12:00Z"
}
```

### Read account statistics

```http
GET /accounts/{account_id}/stats
```

Example response:

```json
{
  "account_id": "acc_123",
  "call_count": 1,
  "total_duration_sec": 143
}
```

## Run locally

Requirements:

- Docker Desktop
- Go 1.25 or later, if running tests directly on the host

Start the service:

```bash
docker compose up -d --build
```

Check health:

```bash
curl http://localhost:8080/healthz
```

Reset local containers and volumes when a fresh database is needed:

```bash
docker compose down -v
docker compose up -d --build
```

## Tests

With Docker Compose running:

```bash
go test ./...
go vet ./...
```

Focused tests cover duplicate delivery, concurrent idempotency, correct statistics, and graceful recording-work shutdown.

## Further details

See [SOLUTION.md](SOLUTION.md) for the root-cause analysis, implementation decisions, scaling considerations, and intentionally out-of-scope changes.
