---
name: kite
description: Use Kite to receive, route, replay, and inspect webhook events from sources like GitHub and Stripe via the `kite` CLI. Covers install, login, streaming, proxying to local servers, manifest-driven routing, and managing endpoints, keys, and the dead-letter queue.
---

# Kite

Kite is a universal webhook adapter. It ingests webhook events from any source (GitHub, Stripe, custom providers), persists them, and lets you stream, proxy, replay, or route them into local dev environments, scripts, or long-running consumers via the `kite` CLI.

Use this skill when the user is integrating with Kite, debugging webhook delivery, wiring a webhook source into a local server, or replaying past events.

This guidance matches Kite CLI `v0.2.2`.

## When to use

- User wants to receive webhooks locally without standing up a public ingress.
- User wants to replay or inspect past webhook events.
- User asks about routing webhooks from multiple sources to different local endpoints.
- User mentions the `kite` CLI, `getkite.sh`, or webhook tunneling.

## Install

```bash
# Checksum-verifying installer (macOS Apple Silicon or Linux x86_64)
curl -fsSL https://getkite.sh/install | sh

# Homebrew for the same supported targets
brew tap alpha-centauri-cyberspace/kite
brew install kite
```

Verify with `kite --version`.

Kite does not publish this CLI to crates.io or as a public container image. The
crates.io package named `kite-cli` is unrelated. For manual installation, use a
checksum-verified asset from the official
[GitHub release](https://github.com/Alpha-Centauri-Cyberspace/kite-cli/releases/latest).

## First-time setup

```bash
kite login                # device-auth flow against https://getkite.sh
kite status               # confirm team + endpoints
```

## Core commands

The CLI is a small set of verbs. Each accepts `--help` for full options.

| Command | Purpose |
|---|---|
| `kite stream` | Stream events to stdout (default summary; `--json` for full CloudEvent, `--compact` for one-liners). |
| `kite proxy` | Forward events to a local HTTP server. Supports single `--target` or per-source `--route SOURCE=URL`. |
| `kite listen` | Receive events via Unix socket (`--socket`) for IPC integrations. |
| `kite run --manifest kite.json` | Run multiple routes / handlers from a declarative manifest. |
| `kite retry --target URL` | Replay events from the dead-letter queue to a target. |
| `kite endpoints` | `list`, `create`, `rotate-secret`, and `deactivate` webhook endpoints. |
| `kite github install --repo OWNER/REPO` | Install a GitHub webhook in-place using local `gh` auth. |
| `kite keys` | `list`, `create`, `revoke` API keys (scopes + expiry supported). |
| `kite logs --limit N` | Inspect persisted event logs. |
| `kite queue` | Inspect and manage the local event queue. |
| `kite agent` | Send / receive agent-to-agent messages on Kite Cloud. |

## Common patterns

### Stream events while developing

```bash
kite stream --source github --json
```

Filters by source, prints full CloudEvent JSON per line — easy to pipe into `jq` or feed into a script.

### Tunnel webhooks to a local server

```bash
kite proxy --target http://localhost:3000/webhooks
```

All sources fan in to one local endpoint. Forwarded requests carry metadata headers:

- `x-kite-source`, `x-kite-event-id`, `x-kite-event-type`, `x-kite-team-id`
- CloudEvents headers: `ce-id`, `ce-source`, `ce-type`, `ce-specversion`, `ce-time`

### Per-source routing

```bash
kite proxy \
  --route github=http://localhost:3001/github \
  --route stripe=http://localhost:3002/stripe \
  --target http://localhost:3000/default
```

`--target` is the fallback. Omit it to require an explicit route per source (unrouted events go to DLQ).

### Install a GitHub webhook into a repo

```bash
kite github install --repo owner/repo
# default events: push, pull_request, issues, issue_comment,
#                 pull_request_review, pull_request_review_comment

kite github install --repo owner/repo --all-events
kite github install --repo owner/repo --events push,pull_request
kite github install --repo owner/repo --rotate-secret
```

Uses local `gh` auth. Idempotent — rerun to update.

### Replay from the DLQ

```bash
kite retry --target http://localhost:3000/webhooks --source github
```

Re-delivers anything that previously failed delivery, optionally scoped to a source.

### Manifest-driven runs

```bash
kite run --manifest kite.json
```

For checked-in routing config that lives alongside the project — preferred over ad-hoc `kite proxy` flags in long-running setups.

## Output modes for `kite stream`

- Default: concise human-readable summary lines.
- `--compact`: one-liner per event, summary only.
- `--json`: full CloudEvent JSON — use this when piping into another program.
- `--exec CMD`: run a command per event with the event JSON on stdin.
- `--client-id ID`: persist a delivery cursor so reconnects resume where you left off.

## Filtering

Source filters narrow WebSocket subscriptions for `stream`, `proxy`, and
`listen`; retry filters apply to the local dead-letter queue. Some output and
importance checks also run locally before delivery:

| Flag | Where | Effect |
|---|---|---|
| `--source <name>` | `stream`, `proxy`, `listen`, `retry` | Only events whose source matches (e.g. `github`, `stripe`). Source is derived from event type — `com.github.push` → `github`. |
| `--event-type <type>` | `stream` | Only events of a specific CloudEvent type (e.g. `com.github.pull_request`). |
| `--importance <level>` | `stream` | Minimum importance: `low`, `normal`, `high`, `critical`. Quiet noisy sources without dropping incidents. |
| `--route SOURCE=URL` | `proxy` | Per-source routing. Acts as a filter when used without `--target` — unrouted sources go to DLQ. |

Examples:

```bash
# GitHub pull-request events only, full JSON for piping
kite stream --source github --event-type com.github.pull_request --json

# Stream only critical events across all sources
kite stream --importance critical

# Proxy only stripe webhooks; everything else gets dropped (no fallback target)
kite proxy --route stripe=http://localhost:3002/stripe

# Replay only failed github events from the DLQ
kite retry --source github --target http://localhost:3000/webhooks
```

Notes:

- Filters compose — `--source` and `--event-type` together AND-match.
- For finer-grained payload filtering (e.g. only PRs into `main`), pipe `--json` through `jq` or use `--exec` and filter inside your handler.
- `--client-id` interacts with filtering: each `client-id` keeps its own delivery cursor, so a filtered consumer with one client-id won't skip ahead for an unfiltered consumer with a different one.

## Authentication model

- `kite login` runs the device-auth flow against `https://getkite.sh` and stores credentials locally.
- Long-lived integrations should use API keys: `kite keys create --name CI --scopes ... --expires-at ...`.
- Revoke with `kite keys revoke --id <ID>`.
- The default server is `https://getkite.sh`; override with `--server` on `login` and `update` for staging or internal test environments.

## Endpoints

A Kite "endpoint" is the addressable URL/credential for a given source.

```bash
kite endpoints list
kite endpoints create --source github --repo owner/repo
kite endpoints create --source linear --signing-secret lin_wh_xxx
kite endpoints deactivate --id <ID>
```

For GitHub, `endpoints create --repo OWNER/NAME` will auto-register the webhook with GitHub (uses `--github-token`, `GITHUB_TOKEN`, or `gh` auth, in that order). `--force` replaces an existing webhook on the repo.

For sources that issue their own signing secret (Linear, Stripe, custom providers), pass it with `--signing-secret <SECRET>`. The secret is encrypted at rest server-side and used to verify HMAC signatures on inbound webhooks. Use `--signing-secret -` to read from stdin (avoids shell history). Re-running `endpoints create` with a new value rotates the stored secret. Without `--signing-secret`, Kite stores no secret for non-GitHub sources and will accept unsigned posts to that endpoint.

## Wire format

Events are delivered as [CloudEvents](https://cloudevents.io/). Source names are derived from the event type — e.g. `com.github.push` → source `github`. Use this when filtering with `--source` or `--route`.

The protocol crate ([`kite-protocol`](https://github.com/Alpha-Centauri-Cyberspace/kite-protocol)) defines the wire format; the CLI pins a compatible minor version. Breaking changes ship as coordinated releases.

## Updating

```bash
kite update              # update in place
kite update --check      # dry-run
kite update --force      # force reinstall
```

Self-update is available only on macOS Apple Silicon and Linux x86_64. It
requires an immutable release-manifest entry and a valid SHA-256 checksum; it
fails closed when either is absent or mismatched.

## Troubleshooting checklist

When events aren't arriving:

1. `kite status` — confirm login + team.
2. `kite endpoints list` — confirm the source has an active endpoint.
3. `kite stream --source <name>` — see if events reach Kite at all.
4. `kite logs --limit 50` — inspect recent persisted events.
5. `kite queue` — check the local queue for stuck items.
6. `kite retry --target <url>` — replay anything in the DLQ.

If events reach `stream` but not your local server, the issue is the proxy/route, not ingestion.

## References

- Product site: https://getkite.sh
- CLI source: https://github.com/Alpha-Centauri-Cyberspace/kite-cli
- Full command reference: https://github.com/Alpha-Centauri-Cyberspace/kite-cli/blob/main/docs/COMMANDS.md
- Protocol crate: https://github.com/Alpha-Centauri-Cyberspace/kite-protocol
