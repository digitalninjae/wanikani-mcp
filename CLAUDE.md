# WaniKani MCP Server — Implementation Plan

## Goal

Build a full-coverage Model Context Protocol (MCP) server, in C# (.NET), that wraps the WaniKani API v2. Unlike the existing community WaniKani MCP servers (all read-only), this one must support **submitting review results** (POST /reviews) so an AI assistant can quiz the user and have correct/incorrect answers actually advance their real WaniKani SRS progress — not just read stats.

Primary consumer: a Claude session teaching the owner Japanese. That session should be able to (a) look up radicals/kanji/vocabulary with official mnemonics, (b) see what's currently due for review, (c) quiz the user, and (d) submit the result back to WaniKani so it counts.

## Tech stack

- **.NET 8** (LTS), C#
- **ModelContextProtocol** NuGet package (official Anthropic/Microsoft-maintained C# SDK) for the MCP server scaffolding (tool registration, stdio and HTTP/SSE transport)
- `System.Net.Http` + `System.Text.Json` for the WaniKani API client — no need for a heavy HTTP client library
- xUnit for tests, with HTTP calls mocked (no hard dependency on hitting the real WaniKani API in CI)

## WaniKani API v2 reference

- Base URL: `https://api.wanikani.com/v2/`
- Auth: `Authorization: Bearer <personal_api_token>` header. Token is obtained by the user from wanikani.com/settings/personal_access_tokens. Support **read-only tokens** as well as full-access tokens — the server should degrade gracefully (return a clear error) if a write is attempted with a read-only token.
- Format: JSON, cursor-based pagination on collection endpoints (`pages.next_url` / `pages.previous_url` in the response), `Update-Since` / `If-Modified-Since` support via `updated_after` query param for incremental sync.
- Rate limit: 60 requests/minute per token. Implement a simple token-bucket limiter in the HTTP client wrapper with backoff on 429.
- Full docs: https://docs.api.wanikani.com/20170710/

### Resources to implement

| Resource | Methods | Notes |
|---|---|---|
| `user` | GET | Profile, current level, subscription status |
| `summary` | GET | What's due now / lessons available (cheap, cached snapshot) |
| `subjects` | GET (list + by id) | Radicals, kanji, vocabulary — includes meanings, readings, mnemonics, component/component-of relationships |
| `assignments` | GET (list + by id), PUT `/assignments/{id}/start` | SRS stage, availability dates, per-item progress |
| `reviews` | **POST** (create), GET (list, deprecated per docs but still useful for history) | The critical write endpoint — submits `assignment_id` (or `subject_id`), `incorrect_meaning_answers`, `incorrect_reading_answers` |
| `review_statistics` | GET (list + by id) | Aggregate correct/incorrect history per subject — useful for leech detection |
| `level_progressions` | GET | Level-up history and timing |
| `resets` | GET | Reset history (informational) |
| `study_materials` | GET, POST, PUT | User's custom notes/synonyms per subject |
| `spaced_repetition_systems` | GET | SRS stage definitions (interval lengths) — needed to interpret `srs_stage` on assignments meaningfully |
| `voice_actors` | GET | Metadata for vocabulary audio — low priority, implement last |

## MCP tools to expose

Read tools (map close to the API, keep names obvious):
- `wanikani_get_user`
- `wanikani_get_summary`
- `wanikani_get_subject` (by id) / `wanikani_search_subjects` (by level, type, slug, or text — this is what lets Claude find "the kanji for 'person'" etc.)
- `wanikani_get_assignments` (filterable: due now, by level, by subject type, by srs stage)
- `wanikani_get_review_statistics`
- `wanikani_get_level_progressions`
- `wanikani_get_study_materials`
- `wanikani_get_srs_systems`

Write tools:
- `wanikani_submit_review` — the core one. Input: subject/assignment identifier + whether the meaning and/or reading answers were correct. Wraps POST /reviews. Return the updated SRS stage and next review time so Claude can confirm to the user in the same feedback-format style already used for kana quizzing (show the item + result, not just "ok").
- `wanikani_start_assignment` — wraps PUT /assignments/{id}/start, needed before an item's first review counts (WaniKani requires lessons to be "started" before review). Claude should call this transparently as part of the lesson flow, not require the user to think about it.
- `wanikani_upsert_study_material` — create/update notes or synonyms.

Every write tool must clearly fail (not silently no-op) if the configured token lacks write permission, with an error message that says exactly that.

## Config / secrets

- WaniKani personal API token via environment variable `WANIKANI_API_TOKEN`. Never log it, never echo it back in tool output.
- No other external dependencies or state — this server is a thin, mostly stateless proxy. Do not build a local database or cache layer beyond in-memory rate-limit bookkeeping; WaniKani is the source of truth.

## Transport / deployment

Support both:
1. **stdio** — for local use with Claude Desktop's MCP config (`claude_desktop_config.json`), simplest for local dev/testing.
2. **HTTP/SSE** — so it can be added as a remote custom connector from a cloud Claude session (Cowork). Needs to run somewhere publicly reachable (a small VPS, Fly.io, Azure Container Apps, etc. — deployment target is up to the user, just make sure the server supports the transport).

Provide a `Dockerfile` for easy deployment of the HTTP/SSE mode.

## Implementation phases

1. **Scaffolding**: .NET project, MCP SDK wired up, health-check tool, config loading, HTTP client wrapper with auth header + rate limiter.
2. **Read path**: implement all GET resources above, with pagination fully handled inside the client (caller-facing tools should return complete result sets or an explicit cursor, not leak WaniKani's raw pagination structure).
3. **Write path**: `wanikani_submit_review` and `wanikani_start_assignment` — the priority feature. Test against a real (sandboxed, low-stakes) WaniKani account before relying on it for real study, since a bad implementation could corrupt real SRS progress.
4. **Study materials**: create/update notes and synonyms.
5. **Polish**: consistent error handling and messages, tool descriptions good enough for an LLM to pick the right one without guessing, README with setup instructions (getting a token, choosing read-only vs full-access, running via Docker or stdio).
6. **Tests**: unit tests for the HTTP client (pagination, rate-limit backoff, auth header) and for each tool's input validation, all against mocked HTTP responses — no real network calls in the test suite.

## Definition of done

- Every resource in the table above has at least a GET tool.
- `wanikani_submit_review` correctly submits both meaning and reading correctness independently (WaniKani tracks these separately for vocabulary/kanji) and returns the resulting SRS stage + next review timestamp.
- Server runs in both stdio and HTTP/SSE modes from the same codebase.
- README documents: obtaining a token, minimum required token permissions per feature, running locally, running via Docker, and adding as a Claude connector (both desktop stdio config and remote HTTP/SSE connector URL).
- No secrets committed; `.env.example` provided.
