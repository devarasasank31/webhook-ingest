# Solution

The duplicate check was a separate `SELECT` followed by an unconstrained
insert. Concurrent deliveries could both observe no event, create duplicate
events, and each increment account totals. Recording work used the request
context after the HTTP handler returned, so its database update was commonly
cancelled; the resulting error was ignored. The work goroutine was also not
tracked during process shutdown.

`events.event_id` is now unique in Postgres, and ingestion uses one database
transaction. `INSERT ... ON CONFLICT DO NOTHING` is the concurrency-safe
decision point: only the transaction that inserts the event upserts the call
and increments durable account statistics. The cache is updated only after
that transaction commits. This keeps duplicate deliveries from producing any
second side effect, and a failed transaction leaves no partial durable state.
Postgres was chosen over Redis for this decision because the event marker and
the related writes need one durable atomic transaction.

Recording work now uses a context independent of the HTTP request, logs
failures with the existing logger, and is tracked with a wait group. Normal
server shutdown waits for work already accepted before closing database
connections. The cache write is also synchronized for concurrent ingestion.

At 10,000 webhooks/second, I would keep the database uniqueness guarantee but
move recording work to a durable queue/outbox with retries and worker
backpressure, partitioning the event workload as needed. I intentionally did
not make Redis part of deduplication, change the HTTP API, or redesign the
existing components.
