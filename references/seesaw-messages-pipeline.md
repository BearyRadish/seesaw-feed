# Seesaw Messages → WeChat Announcements Pipeline

The Seesaw **Messages** surface (teacher announcements/DMs) — separate
from this skill's journal feed. Single owner script:
`~/.hermes/profiles/influencer/scripts/seesaw_announcements_pipeline.py`
(covers ALL Messages content including the Friday newsletter — the old
separate Friday cron is retired).

## Data access — API first (standing rule)

> "All the messages and attachments should be accessible via API. We
> don't need to use browser control to download the file."

Messages has no REST API (all `/api/messages*` guesses 404) but serves
everything over GraphQL:

```
POST https://reloaded-api.seesaw.me/ss2_gql
Authorization: Bearer <token>        # cookies alone → InvalidAuth
```

- **Token**: `localStorage["ASV3-7zeem4"]` on the logged-in SPA tab
  (value is JSON-quoted — parse it). Read via CDP `Runtime.evaluate`.
  Endpoint URLs are published in the SPA HTML bootstrap
  (`window.DYNAMIC_GC_HTTP_URL_v2` / `DYNAMIC_GC_WS_URL_v2`) — re-read
  from the page source when endpoints drift.
- **Query** (unions need inline fragments; edge nodes are `{ message }`,
  NOT `{ node }`):

```graphql
query {
  conversation(input: {conversationId: "<conv.id>"}) {
    ... on ConversationPayload {
      conversation {
        messagesConnection { edges { message {
          id content createDate authorId
          item { id attachment { url sourceUrl } compositeImageUrl shareToken }
        }}}
      }
    }
  }
}
```

- `item.attachment.url` is a SIGNED `assets.seesaw.me/...:::<token>` URL
  — plain `curl` downloads it, no auth headers. Item-card poster images:
  `compositeImageUrl`, same signed format.
- `createDate` is ISO **UTC** — convert +8h (Asia/Shanghai) before
  display or staleness math, or a 2:00 PM Shanghai message reads 06:00.
- Introspection is disabled; discover the schema from validation-error
  suggestions (`Did you mean 'conversationsView'…`). Known query roots:
  `conversation`, `conversationsView`, `conversationsSearch`.

### Hybrid architecture (proven end-to-end)

DOM sidebar scan finds WHICH threads exist (titles, exclusions, unread
badges); the API then loads each thread's messages in ~100ms. Never
build on DOM clicks for content: the "View Original" button's React
wrapper swallows BOTH synthetic `.click()` AND CDP
`Input.dispatchMouseEvent` — DOM-click attachment paths are dead ends.
(The old claim that dispatchMouseEvent counts as trusted input holds for
other elements, not this button.)

### Shape-mismatch pitfall

When normalizing API messages into the pipeline's DOM-shaped dicts,
downloaders must accept BOTH the raw shape (`msg.item.attachment.url`)
and the normalized one (`msg.api_attachment_url`) — a pass-through
layer that reads only one shape fails on a URL that works in isolation.

## Cookie lifecycle

`seesaw-cookies.json` can report `authToken_expired` while the live CDP
Chrome tab still holds a fresh session — re-pull cookies via
`Network.getCookies` from the live tab before concluding a manual login
is needed.

## Thread model (critical)

Each announcement is its OWN thread; teachers have multiple
"Announcement from X" sidebar rows. Dedupe opened threads by conv_id,
never by row title. The sidebar is VIRTUALIZED: scrolling pushes top
rows out of the DOM — delivery scans must NOT scroll (new announcements
sort to the top); only `--seed` scrolls the full list. After any
`location.href` navigation the thread re-mounts asynchronously — poll
for the target message block (up to 15s) instead of a fixed sleep.

## "New message" definition + --unread-only

- New = message ID not in `announcements_state.json` AND ≤ STALE_DAYS
  (2) old AND surfaced by the top-of-sidebar scan. Seesaw's unread badge
  is NOT part of the default definition — a thread read on the phone
  still delivers; the state file is the only dedupe source of truth.
- `--unread-only` (opt-in, default OFF): additionally require the row's
  unread badge. Probe: from the title text node walk up to the row
  container and search ancestors for `[aria-label*="unread" i]` /
  `[data-testid*="unread" i]`.

## Mandatory guards (keep in the script)

1. `--test` → all sends to File Transfer; first runs of any changed
   send pipeline go to Robert only, never the group.
2. Send cap: >2 "new" messages aborts (exit 3) unless `--force`.
3. Re-check sent-state immediately before EACH send.
4. `--seed` marks only messages that carry a date in their timestamp.
5. A 0-message thread open during seeding = red flag; re-run seed.
6. Attachment present but download failed = FAILED send (never degrade
   to text-only); leave unsent for retry next run.

## WeChat send mechanics (WeChat 4.1.60)

- Target group "5CS Class Chat 26-27" pinned at sidebar row 1:
  window-relative (200, 110); File Transfer neutral (200, 175).
- **Neutral row MUST differ from target row** — clicking the same row
  twice DESELECTS it (silent no-op send).
- Every announcement also mirrors to the RP chats (5SW/5FM/5NM/5EP/
  5JA/5JS) with per-target author exclusions — the TARGETS dict in the
  script is the source of truth.
- Text formatting: strip `\u200b`, then collapse 3+ consecutive line
  breaks to exactly 2 (ONE blank line between paragraphs — full collapse
  to single newlines renders too tight).
- Main window non-shareable → sends are blind; verify afterwards
  (activate WeChat, screencapture the window, OCR/read the last bubble).

## State files

- `~/workspace/sas-5cs/announcements_state.json` — message-ID dedupe
  (entries carry author/ts/file/target).
- Downloads: `~/workspace/sas-5cs/announcements/`.

## Exclusion rule

XiaoYan Zhang announcements are for Aspen's Chinese class (G5CLL4)
only — NEVER forward. Filter on author name AND "G5CLL4" title
fragments.

## Cron

`seesaw-daily-announcements` runs the announcements pipeline daily at
17:30 (mirrors ride the same run). Schedule other WeChat-driving crons
outside its ~17-min runtime window (see Pitfall #23 in the main
SKILL.md).
