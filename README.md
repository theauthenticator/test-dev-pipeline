# test-dev-pipeline

Burst parsing capacity for the replay pipeline.

This repo holds a workflow and nothing else. It contains no parser code, no replay data and no
credentials — the worker is checked out at run time from the private `replay-worker` repo, and all
configuration comes from repository secrets.

## How it works

Runners are never told which replays to parse. Each one claims its own work directly from the
database with a single atomic statement that stamps a lease:

```sql
UPDATE processed_sessions SET claimed_by = ?, claim_expires_at = ?
 WHERE rowid IN (SELECT rowid FROM processed_sessions
                  WHERE parsed = 0 AND ... AND (claim_expires_at IS NULL OR claim_expires_at < ?)
                  LIMIT ?)
RETURNING session_id, event_id, window_id, parse_attempts
```

libSQL serialises writes, so two runners can never win the same row. A runner that dies stops
renewing its lease and its replays return to the pool automatically.

That means the runner count is a throughput dial, not a partition — 4 runners and 20 runners both
parse the same backlog correctly, one just finishes sooner. It also means this repo needs no
coordination logic, no work assignment, and no callback endpoint.

## Running it

Actions → **Parse backlog** → **Run workflow**

| input | default | notes |
|---|---|---|
| `runners` | `4` | clamped to 1–20 (free-plan concurrency ceiling) |
| `window_id` | *(blank)* | restrict to one tournament window |
| `worker_ref` | `main` | branch of `replay-worker` to run |

It also accepts `repository_dispatch` with `event_type: parse_burst` and a
`{ runners, window_id }` payload, which is what `burst-dispatch.js` on the VPS sends when a
backlog appears.

## Required secrets

| secret | purpose |
|---|---|
| `WORKER_REPO_TOKEN` | `repo` scope — checks out the private worker |
| `TURSO_DATABASE_URL` / `TURSO_AUTH_TOKEN` | claim work, write status |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | write parsed stats |
| `AWS_DEFAULT_REGION` / `AWS_DYNAMODB_REGION` | S3 and DynamoDB regions |
| `S3_BUCKET` / `DYNAMO_TABLE` | destinations |

## Reading a run

Success is asserted on a `PULL_WORKER_COMPLETE parsed=N failed=M` line, not on the exit code.
The native replay parser holds worker threads that never exit and cannot be terminated without
faulting inside the addon, so the process reliably dies with an access violation during teardown —
after every replay has been parsed, stored and marked. A crash before the drain finishes leaves no
sentinel and still fails the job, so this narrows what counts as success rather than hiding failure.
