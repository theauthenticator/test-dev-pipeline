# test-dev-pipeline

Burst parsing capacity for the replay pipeline.

This repo holds a workflow and nothing else. It contains no parser code, no replay data and no
credentials — the worker is checked out at run time from the private `replay-worker` repo, and all
configuration comes from repository secrets.

## Parser dependency chain

This repo does **not** depend on `@theauthenticator/fortnite-replay-parser` directly.

```text
fortnite-replay-parser  (published package)
        │
        ▼
   replay-worker        (pins + vendors the package in committed node_modules)
        │
        ▼
 test-dev-pipeline      (checks out replay-worker at worker_ref; no npm install of the parser)
```

| Layer | Repo | How the parser arrives |
|---|---|---|
| Source | `theauthenticator` fortnite-replay-parser package | Built and published (GitHub Packages) |
| Worker | [`theauthenticator/replay-worker`](https://github.com/theauthenticator/replay-worker) | `package.json` pin + **committed** `node_modules/@theauthenticator/fortnite-replay-parser` |
| Burst CI | this repo | `actions/checkout` of `replay-worker` at `worker_ref` (default `main`) |

The workflow intentionally skips `npm install` for the parser. Committed `node_modules` already contain the parser, bundled `ffi-napi` prebuilds (including `linux-x64`), and Oodle libraries. Only `@libsql/client` is reinstalled so Linux native bindings match the runner.

### Getting a parser bump into burst runs

1. Publish a new `@theauthenticator/fortnite-replay-parser` version.
2. In **replay-worker**: bump the dependency, reinstall, commit the updated `package.json` / lockfile **and** the vendored `node_modules/@theauthenticator/fortnite-replay-parser` tree (including nested `ffi-napi` / Oodle files), then push the branch you want burst to use (usually `main`).
3. Run this workflow with `worker_ref` pointing at that branch (leave blank / `main` for production).

Bumping only `package.json` in replay-worker is not enough. Burst never resolves the package from the registry; it runs whatever was committed under `node_modules`.
There is intentionally no parser package bump in this repo. The only version selection here is the
`worker_ref` input, which chooses the replay-worker checkout that already vendors the parser.

`npm run deploy` in replay-worker updates the VPS workers. It does **not** update this repo. Burst picks up pull-worker / parser changes only after they are on the `worker_ref` checkout.

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
