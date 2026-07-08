# Copilot Instructions

## Project Overview

Single-file Copilot CLI extension (`extension.mjs`, ~1400 lines) that bridges Telegram messages bidirectionally with a CLI session. It runs as a long-lived process inside the Copilot CLI extension runtime using `@github/copilot-sdk/extension`.

## Architecture

The extension is a monolithic ESM module organized into numbered sections:

1. **Constants & Configuration** — Timing constants, file paths
2. **Utility Functions** — JSON I/O with atomic writes, markdown-to-Telegram-HTML converter
3. **Telegram Bot API Client** — `callTelegram()` wrapper + typed helpers (`sendMessage`, `sendPhoto`, etc.)
4. **Send Queue** — Rate-limit-aware outbound queue with 429 backoff
5. **State Management** — Module-level globals for bot state, session, access control
6. **Access Control & Pairing** — 6-char code pairing flow, auto-lock after first user
7. **Typing Indicator & Tool Bubble** — Ephemeral status messages showing active tool calls
8. **File/Photo Handling** — Download Telegram files to `/tmp`, relay images from tool results
9. **Message Processing** — Inbound Telegram → `session.send()`
10. **Event Handlers** — Outbound session events → Telegram (assistant messages, images, errors)
11. **Slash Commands** — `/telegram setup|connect|disconnect|status|remove|lock|unlock`
12. **Poll Loop** — Long-polling with exponential backoff, 409 conflict handling
13. **Lifecycle** — `joinSession()`, hook registration, SIGTERM cleanup

### Data Flow

```
Telegram user → Bot API (long poll) → processUpdate() → session.send()
session events (assistant.message, tool.*) → Send Queue → Telegram API → user
```

### File Layout at Runtime

```
extension-dir/
  extension.mjs      # The extension code
  bots.json          # Bot registry: { alias: { token, username, addedAt } }
  access.json        # { allowedUsers: [...], pending: {}, locked: bool }
  bots/<name>/
    state.json       # { offset: N } — Telegram poll offset
    lock.json        # { pid, sessionId, connectedAt } — session lock
```

### Key Design Decisions

- **One bot per CLI session** — module-level globals are fine since each process handles one bot
- **Shared access control** — pairing with any bot grants access to all bots
- **Lock-file concurrency** — PID-based stale lock detection; new sessions can take over via Telegram 409
- **Atomic JSON writes** — write to `.tmp` then rename, using `saveJsonAtomic()`

## Conventions

- No build step, no dependencies beyond `@github/copilot-sdk/extension` (provided by the runtime)
- Uses Node.js built-in `fetch` (requires Node 18+)
- All Telegram API calls go through `callTelegram()` which handles timeouts, 409, and 429
- Outbound messages must go through `enqueue()` for rate limiting
- File permissions: `bots.json` saved with mode `0o600`
- Bot names: lowercase alphanumeric with hyphens/underscores only
- The `skills/` directory contains Copilot CLI skill definitions (markdown-based agent instructions)
- The `plugin.json` defines the package for the Copilot CLI plugin registry

## Testing

No test suite exists. Validate changes by:
1. Checking syntax: `node --check extension.mjs`
2. Manual testing with a real Telegram bot (requires `/telegram setup` + `/telegram connect`)

## Important Patterns

- **Hook for token capture**: `onUserPromptSubmitted` intercepts bot tokens pasted during setup (matches `^\d+:[A-Za-z0-9_-]+$`)
- **Markdown conversion**: Custom `markdownToTelegramHtml()` with placeholder-based approach to protect code blocks from double-escaping
- **Bubble messages**: Ephemeral Telegram messages showing tool call status, auto-deleted on `session.idle`
- **Graceful takeover**: When two sessions connect to the same bot, Telegram returns 409 → old session releases cleanly
