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
| `runners` | `4` | clamped to 1–`MAX_RUNNERS` |
| `window_id` | *(blank)* | restrict to one tournament window |
| `worker_ref` | `main` | branch of `replay-worker` to run |
| `target` | *(repo name)* | capacity label; tags `WORKER_ID` so the dispatcher can count this repo's runners separately |

It also accepts `repository_dispatch` with `event_type: parse_burst` and a
`{ runners, window_id, worker_ref, target }` payload, which is what `burst-dispatch.js` on the VPS
sends when a backlog appears.

### Repository variables

Two knobs are variables rather than literals, because this same file is deployed to every capacity
target and only these differ between them.

| variable | default | notes |
|---|---|---|
| `MAX_RUNNERS` | `20` | concurrency ceiling for this repo. 20 is the free-plan cap on hosted runners; raise it for a paid plan or a self-hosted pool |
| `RUNS_ON` | `ubuntu-latest` | set to `self-hosted` (or a label) to run this repo's shards on your own machines, which have no concurrency cap |

## More capacity than one repo can give

Concurrent jobs are capped per account, so a single repo tops out well below what a tournament
backlog needs. Capacity scales by adding more repos holding this same workflow — a second org, a
paid plan, or a repo pointed at self-hosted runners all look identical to the dispatcher.

This is safe precisely because of the claim model above: runners are never assigned work, so
targets need no coordination with each other and cannot duplicate or strand anything. Two repos
with 20 runners each and one repo with 40 parse the same backlog identically.

What ties them together is the `target` label. Each run stamps its shards
`gha-<target>-<run_id>-<shard>`, and `burst-dispatch.js` buckets live leases by that prefix so it
can tell a target that is already saturated from one sitting idle. A target whose label does not
match its entry in `config/burst-targets.json` still parses correctly — it is just invisible to the
sizing logic, which will keep asking it for runners it already has.

To stamp a new target, from the `replay-worker` checkout:

```bash
GH_TOKEN=<PAT for the target account> scripts/provision-burst-target.sh owner/repo 20
```

It creates the repo, syncs this workflow, sets `MAX_RUNNERS`, and reports which of the nine secrets
still need values. It does not copy secret values between accounts — it prints the `gh secret set`
commands and leaves that to you.

A note on where the capacity comes from: spreading across additional **free personal accounts** to
get past the 20-job cap is against GitHub's Acceptable Use Policy, and accounts doing it are liable
to be flagged. Self-hosted runners have no concurrency cap and cost nothing in Actions minutes, and
a paid plan raises the cap directly (Team 60, Enterprise 500). The workflow is identical either
way — only `RUNS_ON` and `MAX_RUNNERS` change.

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
