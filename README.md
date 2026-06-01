# A(I)DEN — Architecture Overview

An autonomous vendor-operations platform built on Anthropic's Claude Agent SDK and Claude Code. Ships to production daily on a single-operator lead-gen marketplace.

This repository is a public architectural overview. Production code lives in a private repository.

## What it does

A(I)DEN runs the full vendor lifecycle for the operator:

- **Inbox triage** — Two-layer Gmail model (filters + shortcuts), label-driven state machine, AI-assisted decisions per thread.
- **Vendor briefs** — Renders the 10-section view of a vendor's current state on demand. Drives every reply.
- **Contract routing** — Detects legal-flavored emails, routes to counsel with required BCCs, tracks in Airtable until signed, files to Box on close.
- **monday.com mutations** — Spawns tasks, flips statuses, swaps groups, updates notes. Dedup before create.
- **Phonexa campaign config** — Active campaign management, tag enforcement, returns-via-ping-back integration.
- **Disposition ingests** — Daily CSV/XLSX ingests from buyers + affiliates into the Phonexa ping-back endpoint.
- **Billing reconciliation** — Inbound invoice matching, returns sheets, credit-ask emails to accounting.

The unifying property: every step touches a structured vendor record. Tomorrow's brief is richer than today's because today's work fed it.

## Architecture at a glance

```
                          +-----------------------------+
                          |    Human operator (Andy)    |
                          |    "next" / "old inbox"     |
                          |    / "send" / explicit ops  |
                          +-------------+---------------+
                                        |
                                        v
              +-------------------------+----------------------------+
              |                Claude Code (runtime shell)            |
              |   - Opus 4.7 holds front-of-house decision context   |
              |   - Sonnet 4.6 subagents fire in parallel for        |
              |     read-only, route-able, codifiable workstreams    |
              |   - 424 machine-readable SOP rules loaded as         |
              |     structured context per session                   |
              +---+-----------------+----------------+----------------+
                  |                 |                |
                  v                 v                v
        +---------+--------+ +------+--------+ +-----+----------+
        | Inbox lane       | | Old-inbox     | | Vendor lane    |
        | (reactive)       | | (drain queue) | | (top-down ops) |
        |                  | |               | |                |
        | newest-first     | | rolling 3     | | brief render,  |
        | triage with      | | prepulled     | | task spawn,    |
        | shortcuts        | | vendors       | | label sweep    |
        +---------+--------+ +------+--------+ +-----+----------+
                  |                 |                |
                  +-----------------+----------------+
                                    |
                                    v
              +---------------------+---------------------+
              |              Tool layer (370+ modules)    |
              +-------------------------------------------+
              | Gmail API (gworkspace.cjs)                |
              | monday.com GraphQL (monday.cjs)           |
              | Phonexa CDP (real Chrome via Playwright)  |
              | Airtable REST                             |
              | Box Sign + Box Drive                      |
              | Google Drive (sync_drive.js)              |
              | Jira REST                                 |
              | OpenAI Whisper (audio transcription)      |
              +-------------------------------------------+
                                    |
                                    v
              +---------------------+---------------------+
              |            State + audit substrate         |
              +-------------------------------------------+
              | monday.com (vendor records, tasks)        |
              | Gmail labels (workflow state machine)     |
              | Box (signed contracts canonical)          |
              | Airtable (in-flight contract tracker)     |
              | Local JSONL audit logs per action class   |
              | Bidirectional Drive sync across 3 hosts   |
              +-------------------------------------------+
```

## The agent topology

A(I)DEN runs three concurrent operational lanes:

### Inbox lane (reactive)

Newest-first triage of the live inbox. Two layers:

1. **Layer 1 — Gmail filters**: known patterns route to `03.noInbox/{category}` automatically. Noise (system notifications, sister-company cross-talk, marketing) never reaches the operator's view.
2. **Layer 2 — Manual shortcuts**: for each remaining thread, the operator (or Claude) picks one of *reply / brief / bury / snooze / forward / ingest*. Each shortcut is one keystroke or one slash-command.

### Old-inbox lane (drain queue)

Vendor-scoped backlog drain. Threads grouped by `zzzVendors/{Vendor}` label; rendered in oldest-thread-latest-message ascending order; one vendor at a time with a rolling 3-vendor prepull pipeline so the next brief is ready by the time the current one ships.

### Vendor lane (top-down)

Periodic per-vendor sweeps independent of inbox state. Catches drift the reactive lane misses: stale `02.waiting/customer` threads, partial-paid invoices, unclosed onboarding template steps, expired snoozes.

## Multi-agent dispatch

When a task has multiple independent subtasks, Claude Code dispatches Sonnet subagents in parallel via the `Agent()` tool. Opus stays as the orchestrator and holds the front-of-house decision context.

Examples of subagent dispatch:

- **Daily audit run** — one Sonnet per buyer board, one per affiliate board, one for Tasks. Three parallel reports stitched together.
- **Vendor brief prepull** — for each of the next 3 vendors in the queue, one Sonnet fetches monday data + Gmail data + helpful links concurrently. Cache lands on disk; the foreground render is instant.
- **Mass label sweep** — one Sonnet per label class, each scoped to its own rule set.

The discipline: subagents are dispatched read-only by default. Write subagents require explicit `allowedTools` whitelist and `isolation: worktree` for any repo edits.

## SOP rules as structured context

424 machine-readable rule files loaded into the session as part of `CLAUDE.md` and topic-file references. Each rule file is a short Markdown doc with:

- A title
- A description (one-line summary for relevance ranking)
- Type (feedback / project / reference / user)
- A body with **Why** and **How to apply** sections

Rules cover:

- Email voice and tone
- Label semantics
- Task lifecycle
- Returns flow per counterparty
- Contract intake routing
- Tool-specific footguns (Phonexa Comment field requirement, Gmail label-search hyphenation, monday API column-format hints)
- AI-assistant operating discipline (reversible vs irreversible, verify after mutation, idle = pick work)

The system "knows" vendor policy the way a trained employee would, with explicit citations from rule body to past incidents that motivated each rule.

## Voice calibration

The Andy-voice model is trained on 4,089 historical sent emails. Outbound drafts pass through a draft-validation gate that scans for:

- LLM tells (em-dashes, "I hope this finds you well", triple-clause sentences)
- Forbidden closings (Best/Sincerely/Cheers/Regards)
- Required closing pair (`Thanks,` then `Andy` on its own line)
- Mojibake / double-encoded characters
- Missing legal-team BCC when subject/body triggers legal-flavor keywords
- Coworker ownership (if a coworker has more outbound on the thread, throw)
- Prior outbound to the same recipient in the last 48h on a different thread

Drafts that fail the gate are blocked at staging time, never reach send.

## CDP automation

Where APIs don't exist or are too slow, Chromium DevTools Protocol drives a real Chrome instance (not Playwright's bundled Chromium — clean flag set, no automation tells):

- Phonexa Lead Index reports (CDP + DOM scrape; lazy session refresh)
- Phonexa campaign edits (form fill + comment-field guard)
- Gmail snooze (CDP click on the `uUQygd` button at the right coords)
- Box Sign envelope status per-signer

A global concurrency lock + size-ordered queue ensures one CDP job per machine at a time, since Cloudflare bot-score is per-IP not per-port. Smallest job runs next (report pulls before deep crawls).

## Bidirectional Drive sync

Operator runs A(I)DEN from three physical machines (home desktop, work laptop, Linux VM). State sync via `sync_drive.js`:

- Drive folder `claude-code-state` is canonical
- Home pulls at 4pm + 5am
- Linux VM pulls at 5am + 11pm + on reboot
- Work laptop pushes at noon (push-only, known one-sided sync gap, kept intentional)

Fork-safety: pull before edit, push after edit, don't concurrently touch MEMORY.md across sessions. Multi-session sync rule loaded as structured context so Claude doesn't step on its own state across machines.

## Production discipline

Three patterns that make A(I)DEN survive a bad week:

1. **Verify after mutation.** Every state-flipping POST/PUT/DELETE is followed by a read-back. A 200 OK is not proof of state change.
2. **Background by default.** Long-running work (subagents, scrapes, audit sweeps) runs in the background. Main thread stays responsive for the operator.
3. **Idempotent helpers, not scratch scripts.** No freelance `_scratch_*.cjs` for mutations. Use the tested helpers (`gworkspace.cjs`, `monday.cjs`, `post_send_labels.cjs`) which encode the operating rules.

## What's public

Patterns and tooling that survived A(I)DEN's daily production grind, extracted as standalone projects:

- [vendor-ops-playbook](https://github.com/andy-bttryzn/vendor-ops-playbook) — the operating manual
- [vendor-brief-renderer](https://github.com/andy-bttryzn/vendor-brief-renderer) — the 10-section brief renderer
- [gworkspace-helper](https://github.com/andy-bttryzn/gworkspace-helper) — the Gmail helper with draft validation
- [monday-helper](https://github.com/andy-bttryzn/monday-helper) — the monday.com helper
- [docs-mirror-scraper](https://github.com/andy-bttryzn/docs-mirror-scraper) — the offline-docs crawler

## What's private

The orchestrator, the SOP rule corpus, the Phonexa CDP layer, the Gmail-side handlers, the cron + watcher daemons, the bidirectional sync logic, voice-calibration training data. Happy to walk through architecture in conversation.

## Contact

andy@bttryzn.com
