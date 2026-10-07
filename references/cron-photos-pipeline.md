# Cron pipeline: 8PM Mon-Fri Seesaw photos → WeChat

End-to-end recipe for the cron job `40ef4787548e` "5CS Seesaw Photos → WeChat
(8PM CST, Mon-Fri)" that fires `0 20 * * 1-5` (LOCAL CST). Verified working
on 2026-09-04 (creation + manual test run). This file documents the
companion script and the cookie-lifecycle pattern that lets the cron run
unattended even though Seesaw forbids automated login.

## Components

| File | Purpose |
|---|---|
| `~/.hermes/profiles/influencer/scripts/seesaw_photos_pipeline.py` | Deterministic helper: cookie validity check, today's-posts filter (excludes Aspen), metadata writer, photo download. Called by the LLM agent; never imports browser-only modules. |
| `~/.hermes/profiles/influencer/seesaw-cookies.json` | Seesaw auth cookies (9 cookies, ~4KB). Saved after a successful manual login. Read by the helper script; written by the LLM agent via `cdp('Network.getCookies', urls=['https://app.seesaw.me/'])`. |
| Cron `40ef4787548e` | LLM-driven cron (NOT `no_agent`). Loads `seesaw-parent-browser` + `wechat-macos-automation` skills. Calls the helper script via `subprocess.run` for the deterministic parts; does the browser + WeChat GUI parts inline. |

## The pipeline script: `seesaw_photos_pipeline.py`

Three modes, all designed to be called via `subprocess.run` from a
`browser_exec` block:

### `--check-cookies`

Validates the saved cookies. Exit code 0 = valid, exit code 1 = expired.
The cron's first step is:

```bash
python3 ~/.hermes/profiles/influencer/scripts/seesaw_photos_pipeline.py --check-cookies
```

If exit code is non-zero, the cron sends ONE friendly Feishu message
("please log in once at https://app.seesaw.me/ and reply 'logged in'")
and stops. No automated login attempts (Seesaw rate-limits aggressively).

**Validity gate**: the script checks the `authToken` cookie's `expires`
field against current time. If `expires < now`, the entire session is
considered dead. Other long-lived cookies (Google OAuth state, AMP
analytics) don't matter — `authToken` is the actual session token.

### `--extract-today`

Reads feed metadata JSON from stdin, filters to today's posts authored by
teachers (drops "Aspen Hsiung" per the family privacy rule), and prints
the filtered list as JSON. Empty `teacher_posts_today` array means cron
exits silently with no chat message.

Input format (what the cron passes on stdin):

```json
{
  "posts": [
    {"caption": "...", "author": "Ms. Sitler" | "Aspen Hsiung" | "Everyone",
     "date": "Sep 4, 2026 11:17 AM", "folders": [...], "carousel_count": 7,
     "source_url": "https://app.seesaw.me/#/family/feed"}
  ]
}
```

The author normalization rule is important: Seesaw labels teacher posts
as "Everyone" (the default class-wide visibility tag) and student posts
as the student's own name. The cron prompt normalizes "Everyone" to the
teacher's actual name (looked up from `folders` or "Pinned by Teacher"
text) before passing to the script — or, for simplicity, the script
treats both "Everyone" and any teacher name (Ms. Sitler, Mr. Limpisvasti)
as INCLUDE, and explicitly checks for "Aspen Hsiung" as EXCLUDE.

### `--save-cookies`

Documented but not yet implemented as a script subcommand. The cookie
saving happens inside the LLM agent's flow after a successful manual
login:

```python
result = cdp('Network.getCookies', urls=['https://app.seesaw.me/'])
import json
with open('/Users/openclaw/.hermes/profiles/influencer/seesaw-cookies.json', 'w') as f:
    json.dump(result['cookies'], f, indent=2)
```

The full cookie list (9 cookies) is ~4KB. Only the `authToken` matters
for subsequent runs; the other 8 are analytics/Google-OAuth-state that
happen to be set.

## The cron prompt pattern

The cron's full prompt is ~150 lines and lives in
`/tmp/seesaw-photos-cron-prompt.md` on the local machine. Key sections:

1. **Cookie check** — `python3 seesaw_photos_pipeline.py --check-cookies`.
   Exit 0 → proceed. Exit 1 → friendly reminder + STOP.
2. **Login if needed (visible Chrome + CDP via `seesaw_cdp_login.py`)** —
   only runs when cookies are absent. Polls up to 12s for the React
   role-picker / password input; on every poll also probes for reCAPTCHA
   (`iframe[src*="recaptcha"]`, `.g-recaptcha`, body text matching
   `captcha|robot|verify you|unusual traffic`). **If captcha detected at
   any point**, the script exits with **code 6** and reason
   `captcha_required`. STOP and surface clearly to Robert — no automated
   retries, no headless fallback. Robert opens the visible Chrome window
   at :9222, completes the captcha, then says "retry" or waits for
   tomorrow's run. (See skill Pitfall #22 for the rationale; the
   `time.sleep(6)` retry was a silent-failure pattern that masked
   captcha triggers.)
3. **Open Seesaw feed** — `new_tab("https://app.seesaw.me/#/family/feed")`,
   verify URL is `/#/family/feed` (NOT `/#/login`).
4. **Extract today's posts** — single JS query (see
   `references/scrollable-feed-pattern.md` for the JS pattern).
5. **Filter posts** — pipe to `seesaw_photos_pipeline.py --extract-today`.
   Empty result → exit silently.
6. **Per-post photo extraction** — for each filtered post, click into
   lightbox, iterate via right arrow (~1713, 348 viewport coords), capture
   each `naturalWidth >= 1500` image URL.
7. **Download photos** — use `urllib.request` to fetch each URL (signed
   tokens are valid for the session). Save to
   `~/workspace/sas-5cs/seesaw-photos/<date>-<slug>/photo-NN-<uuid>.jpg`.
   Write `metadata.json` with the post caption, author, date, folders,
   and photo list.
8. **WeChat send** — open WeChat file picker, navigate to folder,
   **rename `metadata.json` → `.metadata.json` BEFORE the picker opens**
   so Cmd+A selects only the photo file (NSOpenPanel hides dot-prefixed
   files by default), click merge bubble at (363, 960) to send as one
   merged message, then rename `.metadata.json` → `metadata.json` back in
   a `finally:` block so other scripts (`--compose-message`) can still
   read it. See `wechat-macos-automation` Pitfall #55 for the rule and
   `send_5cs_seesaw_photo.py::navigate_picker_to` for the verified
   implementation. The picker-hygiene step is mandatory — Robert
   explicitly does not want `metadata.json` riding alongside the photo
   to the parent group.

   **The proven send recipe** lives in
   `seesaw_announcements_pipeline.py::wechat_send_file_and_text()` and
   is the ONLY verified path to deliver a photo+text pair on this Mac
   mini. Two ways to use it:
   - **Call it directly** (preferred): import
     `seesaw_announcements_pipeline.wechat_send_file_and_text` and pass
     `(file_path, text, target="group")`. The function acquires the GUI
     lock, opens WeChat, selects the target chat, stages the file in a
     fresh temp dir (prevents the Cmd+A-sweeps-stale-files bug from
     2026-09-07), drives the picker with the proven click sequence
     (window-relative folder-icon `(x+310, y+937)`, Cmd+Shift+G "Go to
     folder", Cmd+A, Return to open, Return to send), then pastes the
     text and presses Return.
   - **Mirror the recipe verbatim** if calling the function is
     impossible (e.g. outside an LLM agent context). The click sequence
     uses **window-relative** coords derived from the live window
     origin (`osascript position of window 1`), NOT screen-coord
     guesses — guessing the icon position from screenshots is a
     silent-failure pattern (Pitfall #78 in `wechat-macos-automation`).

   **Verification after each click** is mandatory, not optional. After
   the folder-icon click, take `screencapture -x`, vision_analyze for
   "is the file picker visible?" — if not, the click missed and the
   subsequent keystrokes will land in the selected chat's input field
   (a separate silent-failure class). After the photo-send Return,
   vision_analyze for "does the chat history show a new photo bubble?"
   before continuing to the text companion — a `screencapture` showing
   the chat unchanged after a `[done]` log is the exact failure
   signature of "shell script fired all clicks without any landing."
9. **Final report** — send ONE summary Feishu message with N teacher
   posts, M photos extracted, ✅/❌ WeChat status.

## Cookie lifecycle (verified 2026-09-04)

| Phase | What happens | Action required |
|---|---|---|
| **First run after cron creation** | No cookies saved. Cron exits with friendly reminder. | Robert opens Chrome, logs in once, replies "logged in". Agent saves cookies. |
| **Subsequent runs (next 1-7 days)** | Cookies valid. Cron runs end-to-end. | None. |
| **Cookies expired** | `authToken` `expires` is in the past. Cron sends reminder, stops. | Same as first run — Robert re-logs in. |
| **Seesaw forces re-auth mid-session** | URL redirects to `/#/login` despite cookies. Cron treats as expired, sends reminder. | Same as first run. |

**Expected refresh cadence**: every 1-7 days depending on Seesaw's
sliding-window behavior. Not a daily task — only when the cron reports
expiry.

## Single-photo delivery strategy (Robert 2026-09-22)

When a Seesaw teacher post has more than one photo, **send only the first photo to the 5CS group, then a text message pointing to the rest on Seesaw**. This replaces the previous "merge all photos into one bubble" approach. The pipeline script (`seesaw_photos_pipeline.py`) was refactored accordingly — see the section "What the script now does" below.

### Why single-photo + "more on Seesaw" (not chunked merges)

- **WeChat hard-caps a single message at 9 attached files** (see `wechat-macos-automation` Pitfall #75). Any post with >9 photos is impossible to deliver as a single merged bubble; chunking 9+9+1 leaves 3 separate bubbles that arrive out of order with the caption somewhere in the middle — parents have to scroll-reassemble.
- **Group-chat spam posture is "≤9 per send" but the new floor is much lower.** Even a 5-photo album arrives as a wall of one-per-row preview thumbnails that obscure the chat for parents reading later. Robert's directive 2026-09-22: "only send the first image and say how many more are on Seesaw." The trade-off is intentional — fewer parents click the album, but those who do see all of it on Seesaw's proper gallery.
- **The text message is the link.** Parents get a real caption ("Jennie Chen, Leon Jiang, Max Zhou practicing partner throwing"), one preview line per skipped photo with student names truncated to ~40 chars so other kids get surfaced, then `📸 N more photo(s) available on Seesaw:` and the post permalink. The preview lines are the deliberate incentive to click through — without them the link sits alone and gets ignored.
- **The local archive stays lean.** With single-photo delivery, the cron no longer needs `--all-photos`. Pass `--first-photo` (or omit `--all-photos` once the script's default is updated) so the only photo saved locally is the one we send. Future sessions investigating the cron don't find stale archives of multi-photo albums.

### What the script now does

`seesaw_photos_pipeline.py` gained a new helper `compose_wechat_message_with_more_photos(meta)` that renders the text-bubble body:

```
<first photo's caption or teacher_comment / post fallback>
— <author> (<date>)

• <skipped photo 2 caption, truncated to 40 chars>
• <skipped photo 3 caption, truncated to 40 chars>
• <skipped photo N caption, truncated to 40 chars>

📸 N-1 more photo(s) available on Seesaw:
🔗 <permalink>

💬 Parents: please open Seesaw and leave an encouraging comment for your child!
```

The legacy `compose_wechat_message` (single-bubble, no preview, no "more" line) is preserved as `--compose-message-legacy` for diffing against old behavior. The default `--compose-message` now uses the new format.

Per fresh post: **2 WeChat bubbles total** (1 photo + 1 text). Older posts without per-image captions still flow through the new function — `image_captions` is empty, so the body falls back to `teacher_comment` / `post`, and `skipped_count` is 0 so the "N more" line is omitted. The text bubble just becomes "caption + permalink + closer" with no preview.

### Cron-prompt changes (jobs.json entry for `40ef4787548e`)

Step 5 in the prompt was rewritten to mandate single-photo delivery for ALL posts (per-image or merged — unified). Step 6 ("Merged send strategy") was deleted; the old steps 7-8 became 6-7. The pipeline integrity invariants are unchanged: 7-day age gate (`seesaw_backfill.py --unsent-fresh`), marker-exclude Aspen, GUI lock, the user's "test on X = only X" rule (`wechat-macos-automation` Pitfall #77).

### Verify the new format before unpausing

Until `40ef4787548e` is unpaused, dry-run the new helper against any real folder in `~/workspace/sas-5cs/seesaw-photos/<date>-*/`:

```bash
python3 ~/.hermes/profiles/influencer/scripts/seesaw_photos_pipeline.py \
  --compose-message <folder>
# vs. legacy for diff:
python3 ~/.hermes/profiles/influencer/scripts/seesaw_photos_pipeline.py \
  --compose-message-legacy <folder>
```

The legacy output has 3 lines (body, author line, permalink + closer); the new output adds preview bullets + the "📸 N more on Seesaw" line. Both should look right; only send if the new format reads clean.

## Why LLM-driven, not `no_agent`

`no_agent=true` is right for digest scripts that just process text (the
existing `seesaw_announcements.py`, `unified_digest.py`, etc.). The
photos cron is different because it needs:

- **Browser vision verification** — `vision_analyze` on screenshots to
  confirm lightbox state, post content, merge bubble color
- **Interactive error recovery** — if a click misses, re-derive coords;
  if a post has unexpected layout, adapt
- **Multi-skill orchestration** — `seesaw-parent-browser` (DOM extraction)
  + `wechat-macos-automation` (file picker + merged send)

A `no_agent` script could theoretically do all this, but the boilerplate
for vision verification + retry loops would dwarf the actual work. The
LLM agent handles the visual/tactical parts inline.

## Failure modes the cron must handle gracefully

- **No teacher posts today** → exit silently, no chat message
- **Cookies expired** → friendly reminder, no further work
- **Single post fails to extract** → log error, continue to next post
- **WeChat send fails** → photos are saved locally, log error, continue
  to next post. Don't retry WeChat endlessly.

## Manual verification recipe

To test the cron end-to-end without waiting for 8 PM:

```bash
hermes cron run 40ef4787548e --prompt "Manual test — verify today's pipeline.
Today (2026-09-04) has only Aspen's post; expected outcome is
'exit silently, no chat message'. Confirm cookies valid + browser session
works + filter correctly excludes Aspen. Fix any bugs in
~/.hermes/profiles/influencer/scripts/seesaw_photos_pipeline.py
and report. Otherwise just exit silently per the prompt."
```

The cron runs in the background; the result re-enters the conversation
when complete (typically 2-5 minutes).

## Reference files

- `~/.hermes/profiles/influencer/scripts/seesaw_photos_pipeline.py` — the helper
- `~/.hermes/profiles/influencer/seesaw-cookies.json` — saved cookies
- Cron `40ef4787548e` — runs at 8 PM Mon-Fri
- `references/scrollable-feed-pattern.md` — JS recipe for full-feed post
  extraction (used by the cron's step 3)
- `../wechat-macos-automation/SKILL.md` — WeChat send workflow with merge
  bubble at (363, 960)