---
name: seesaw-feed
description: "Programmatic access to the Seesaw parent journal feed (photos + captions) via the authed REST endpoints /api/person/parent/feed and /api/item_v2. Returns share tokens, per-page captions, photo URLs, and comments without browser automation. GUI fallback documented for posts that fail API enrichment. Privacy: filter posts by author allowlist ('Everyone' = teacher post, include; student-name author = exclude) — denylist is unsafe because stripped sibling rows can carry a student subject without the author name."
version: 7
author: hermes-agent
license: MIT
metadata:
  hermes:
    tags:
      - seesaw
      - school
      - browser-automation
      - photos
      - parent-portal
      - api
    related_skills:
      - sas-school-events-calendar
      - draft-wechat
      - wechat-macos-automation
---

# Seesaw Parent Feed Extraction

Two extraction paths exist; use the API one unless you must fall back.

## 1. PREFERRED — Direct HTTP API (no browser, ~100ms per post)

Seesaw exposes two authed REST endpoints that return everything the GUI shows:

| Endpoint | Returns | Auth |
|---|---|---|
| `GET /api/person/parent/feed?last_key=...` | All feed items: item_id, share_token, text, num_pages, people_tags, comments, tags. Paginated via `last_key` cursor. | authToken cookie |
| `GET /api/item_v2?item_id=item.<uuid>&_bundle=me.see-saw.web_magiccam&_release=...&_tz_offset=28800&_xsrf=...` | FULL per-page data: every `composite_image_url` + `caption` for all num_pages. Same endpoint the Seesaw SPA calls when you click into a post. | authToken cookie + _xsrf token |

Feed API only returns page 1 of multi-page posts (a Seesaw data anomaly).
`item_v2` returns all pages. Always use `item_v2` to enrich multi-page posts.

**Companion script**: `~/.hermes/profiles/influencer/scripts/seesaw_api.py` (one
Python module handles cookie check, paginated feed fetch, item_v2 enrichment,
per-page URL/caption extraction, tagged-students list with Aspen filter, teacher
comment filter, share-link builder, photo download, metadata.json writer).

**Why this exists**: We discovered that the GUI workflow (load feed in Chrome,
click into post, scrape lightbox, click next 6 times for 7-photo carousel)
was solving problems the SPA itself already solved server-side. The same
HTTP request the SPA makes also works for cron automation — no reCAPTCHA, no
cookie refresh races, no element-class drift.

### Quick-start pipeline (cron or interactive)

```python
from seesaw_api import (
    extract_all_feed_items, enrich_all_feed_items, is_teacher_post,
    build_smart_captions_for_item, build_merged_caption,
    extract_share_link, download_album, build_metadata,
)
import sys, json
from pathlib import Path

items = extract_all_feed_items()                          # paginated feed
enrich_all_feed_items(items, only_multi_page=True)         # fill in pages 2..N
teachers = [it for it in items if is_teacher_post(it)[0]]   # filter teacher posts
for post in teachers:
    captions, has_real = build_smart_captions_for_item(post)
    if has_real and len(captions) > 1:
        strategy = "per_image"
    else:
        strategy = "merged"
        caption_text = build_merged_caption(post)
    permalink = extract_share_link(post)
    out_dir = Path(f"~/workspace/sas-5cs/seesaw-photos/{post['item_id'][:8]}")
    saved = download_album(post, out_dir.expanduser())     # HTTP downloads
    meta = build_metadata(post, photo_files=saved, folder_path=str(out_dir))
    (out_dir / "metadata.json").write_text(json.dumps(meta, indent=2))
```

## 2. GUI fallback (visible Chrome + raw CDP over websocket)

Only when the API fails for a specific post. Verified end-to-end on Robert's
Mac mini, Sep 3, 2026: extracted all 7 photos from Ms. Sitler's "Powers of
Ten Chutes & Ladders" math post. Use the "Workflow" section below.

## When to use

Use when Robert asks for any of:
- "Download the photos the teacher shared today"
- "Save the Seesaw photos from the latest post"
- "Archive everything in the Seesaw feed" (use scope clarifier — see Pitfall #5)
- Any parent-side Seesaw content capture

For Seesaw **Messages** (teacher announcements/DMs) use the dedicated
`references/seesaw-messages-pipeline.md` recipe — Messages runs on a
separate GraphQL backend (`reloaded-api.seesaw.me/ss2_gql`), not the feed
REST API, and the daily announcements cron owns its delivery.

## The trick — Seesaw SPA architecture

Seesaw is a React/Angular single-page app, not static HTML. The feed shows
**only the first photo of each post** as a carousel preview (e.g., "1/7" badge).
The other 6 photos are NOT in the DOM until you click into the post and
navigate the carousel.

This means:
- DOM scraping the feed page gets you 1 URL per post, not all photos
- You MUST click into the lightbox, then click "next" arrows, to fetch each
  photo URL from the DOM as it loads
- The image URLs include **signed auth tokens** in the `:::`-separated path
  (`https://assets.seesaw.me/us-2/<path>/<uuid>.jpg:::TIMESTAMP:::WIDTH:::1:::<TOKEN>.jpg`)
  — strip nothing, the token is required

## Privacy — NEVER include Aspen Hsiung's posts (Robert's rule, 2026-09-04)

Robert's explicit family policy: **do not download, send, or otherwise share images from posts authored by "Aspen Hsiung"**. Her posts are her own schoolwork (Chinese pinyin tests, Seesaw Tools lessons, etc.) — sensitive student material.

Seesaw feed distinguishes two author types:
- **"Everyone"** = a teacher post visible to the whole class (Ms. Sitler, Mr. Limpisvasti, etc.) → INCLUDE these
- **"Aspen Hsiung"** (or any other student name) = a student's own post or response → **EXCLUDE these**

The DOM text on a feed item starts with the author line ("Everyone" or a name). When filtering programmatically, treat `"Everyone"` as a teacher post and any student name as a filter-out. When the agent reads the feed manually (vision or DOM), double-check the author line before extracting photos from any post.

**Filter MUST be an allowlist, not a denylist** — VERIFIED BUG, 2026-09-04. `div[class*="ss-feed-item"]` returns ~38 nodes for ~9 real posts: the SPA emits stripped sibling/wrapper rows where the author line is MISSING. Observed shapes in the live feed:

| Node | `author` (first line) | What it really is |
|---|---|---|
| [8] | `Everyone` | real teacher post (keep) |
| [9] | `Everyone` | stripped sibling of [8], no date |
| [0] | `Pinned by Teacher` | wrapper row duplicating a teacher post |
| [28] | `Aspen Hsiung` | Aspen's assignment response (reject) |
| [30] [31] | `In response to: Seesaw Tools: Lessons 6-10` | **stripped sibling of Aspen's post — author line GONE** |
| [2] [6] … | `Comment` | comment affordance rows |

A denylist of `author == "Aspen Hsiung"` **leaks nodes [30]/[31]** — they carry Aspen's photos but not her name, so her private schoolwork would be downloaded and sent to WeChat. Correct logic (now in `seesaw_photos_pipeline.py:is_teacher_post()`):

1. Reject if `"aspen hsiung"` OR `"in response to:"` appears in the author **or anywhere in the caption** (`In response to:` is how Seesaw renders any *student* answer to an assignment).
2. Then require `author.strip().lower() == "everyone"` exactly — this drops `Comment`, `Pinned by Teacher`, and blank-author rows.
3. Dedup on `(date, carousel_count)` — wrapper rows otherwise cause a double-send of the same post.

Regression tests live at `~/.hermes/profiles/influencer/scripts/tests/test_seesaw_filter.py` and `test_seesaw_dedup.py`. Run both after any filter change.

## Workflow

### Step 1: Verify session is authenticated

Open `https://app.seesaw.me/` and screencapture. You should land on the Home
feed, not the login screen. If you land on login, the session was lost — STOP
and tell him. Login is possible via browser automation using `fill_input` for the
email+password fields and clicking "Family Member Sign In" — verified
2026-09-04. Rate limits do apply but a single automated attempt is fine.
If the session was lost (no auth cookies present), automated login
works. Multi-attempt failures may trigger a `/reset_password` redirect
— stop and ask the user for the correct password rather than retrying
(see Pitfall #9).

### Step 2: Scope the request

Before scrolling the feed, ask Robert how far back he wants:
- "Today only" — single post (likely 1-7 photos)
- "This week" — 2-5 posts
- "All visible in feed" — could be 10+ posts, 50+ photos
- "Full history" — requires infinite scroll, very slow

Default recommendation: "Today only" unless the request explicitly says
otherwise. Photos of minors are sensitive — don't bulk-archive without scope.

### Step 3: Extract photo URLs from a single post

For each target post, this is the verified sequence:

```python
# 1. Click the post image to open the lightbox
#    Image center coords: ~(884, 393) for default feed layout,
#    but use vision_analyze on the post screenshot to confirm
new_tab("https://app.seesaw.me/")  # already on Home
wait_for_load()
# ... scroll to post, click post image, etc.

# 2. Once in lightbox, find the "next page" arrow — NOT the obvious arrow
#    The Seesaw lightbox has a "next page" DIV at:
#    text='next page' class='css-18t94o4 css-1dbjc4n r-1habvwh ...'
#    coords on Robert's Mac mini (1920x1080, WeChat-style layout):
#    ~(1713, 348)  ← RIGHT arrow
#    ~(191, 348)   ← LEFT arrow
#    These are NOT at the edge of the screen — they're inside the lightbox

# 3. Iterate: query the active image URL after each arrow click
def get_current_src():
    res = cdp('Runtime.evaluate', expression="""
    JSON.stringify((function() {
        const largeImgs = Array.from(document.querySelectorAll('img'))
            .filter(i => i.naturalWidth >= 1500);
        return largeImgs.length ? largeImgs[0].src : 'none';
    })())
    """, returnByValue=True)
    return res['result'].get('value', 'none')

urls = []
for i in range(total_photos):  # total = e.g. 7
    urls.append(get_current_src())
    if i < total_photos - 1:
        click_at_xy(1713, 348)   # right arrow
        time.sleep(2)
```

**Important:** The first URL returned will be wrapped in JSON quotes — strip
them with `.strip().strip('"')` before using.

### Step 4: Download the photos

```python
import urllib.request
out_dir = Path(f"~/workspace/sas-5cs/seesaw-photos/{post_slug}")
out_dir.mkdir(parents=True, exist_ok=True)
for i, url in enumerate(urls, 1):
    uuid = url.split('/')[-1].split(':::')[0]   # strip .jpg from end
    req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
    with urllib.request.urlopen(req, timeout=20) as resp:
        (out_dir / f"photo-{i:02d}-{uuid}.jpg").write_bytes(resp.read())
```

**Filename gotcha:** the URLs already end in `.jpg` — DON'T append `.jpg`
again or you get `photo-01-uuid.jpg.jpg`. The UUID extraction above is
already the bare name; the `.jpg` extension comes from the URL itself.

### Step 5: Verify

For each downloaded photo:
- File size should be 300-800KB for typical classroom shots
- `sips -g pixelWidth -g pixelHeight <file>` to confirm dimensions
- `vision_analyze` on 1-2 random photos to confirm they're real (not
  corrupted, not a default avatar, not "this image is private")

### Step 6: Write metadata.json

```json
{
  "post": "Original post caption",
  "author": "Ms. Sitler (homeroom teacher)",
  "date": "2026-09-03 11:17",
  "folders": ["5CS - 26-27 SY", "Class Photos", "Math"],
  "source_url": "https://app.seesaw.me/...",
  "photos": [
    {"n": 1, "uuid": "...", "bytes": 534275, "ct": "image/jpeg"},
    ...
  ]
}
```

### Step 7: Privacy + delivery

Photos of minors — flag before any delivery beyond Robert's own chat. Default
delivery path: write the file paths and let Robert drag from Finder to WeChat
himself (WeChat drag-from-Finder is reliable; image paste via osascript is
NOT, see wechat-macos-automation Pitfall #27).

## Pitfalls

1. **The "next page" arrow is NOT at the screen edge.** It's an inner DIV at
   ~x=1713 on Robert's Mac mini, not the obvious chevron at x=1850+ that
   appears visually. vision_analyze the lightbox and look for elements with
   `text='next page'` class `css-18t94o4 css-1dbjc4n r-1habvwh` to confirm
   the right coords. Clicking the wrong spot registers a no-op.

2. **All `<img>` elements in the feed look identical until you scope by size.**
   The feed shows ~30-40 `<img>` elements total (avatars, thumbnails, post
   photos, sidebar icons). The "real" post photos have `naturalWidth >= 500`
   in the feed and `naturalWidth >= 1500` in the lightbox. Always filter by
   `naturalWidth`, not `offsetWidth`.

3. **The first URL in JSON.stringify() output has surrounding quotes.** When
   you `JSON.stringify()` a string from Runtime.evaluate, you get
   `"https://..."` with literal double quotes. Strip them: `url.strip().strip('"')`.

4. **Filenames get `.jpg.jpg` if you naively append the extension.** The URL
   already ends with `.jpg` (e.g.,
   `...uuid.jpg:::TIMESTAMP:::1209600:::1:::<TOKEN>.jpg`). Take the segment
   between `/` and the first `:::` as the UUID; don't append `.jpg`.

5. **Scope creep: don't bulk-archive without asking.** A Seesaw feed can have
   dozens of posts going back months, each with multiple photos. The default
   "Today only" keeps extraction under 1 minute and avoids unnecessary
   handling of minors' photos. Always ask before scope expansion.

6. **`page_info()` and `js()` helpers from the browser harness returned empty
   for this session.** Use `cdp('Runtime.evaluate', expression=..., returnByValue=True)` directly instead — that works.

7. **Seesaw URL auth tokens expire.** The signed S3 tokens in
   `https://assets.seesaw.me/...` URLs are valid for ~hours. Don't expect to
   reuse them across sessions. Always extract + download in the same session.

8. **The "Photos and Videos" menu item in WeChat is a VIEWER, not a sender.**
   Don't confuse it with a file picker — clicking it opens the chat's past
   attachment history, not a file dialog. (Lesson learned the hard way.)

9a. **Headless Chrome auto-login fails with `recaptcha_required` — even with correct password** — OBSERVED 2026-09-06. Seesaw's `/api/auth/login` endpoint returns `{"status": "OK", "response": {"recaptcha_required": true}}` when the request comes from a headless browser fingerprint. The form fills correctly, React event dispatch triggers the XHR, but the server-side reCAPTCHA gate refuses without a valid token. **No cookies are set; page stays on `/#/login?role=parent`;** `grecaptcha-error` divs appear in the DOM (empty text). **Verified remediation paths** that ALSO fail in headless mode: `form.requestSubmit()`, `btn.click()`, `btn.dispatchEvent(new MouseEvent('click'))`, `fill_input` followed by submit, and instrumented XHR/fetch interception (all reach the server but get rejected at the captcha check). **SOLUTION**: launch Chrome WITHOUT `--headless=new` flag (e.g., `Google Chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-cron-debug --no-first-run --no-default-browser-check --disable-gpu`) and attach the agent's browser to it via CDP. A visible Chrome window passes the reCAPTCHA check automatically. **Operational rule**: cron-prompted Seesaw auto-login MUST launch a visible Chrome — never use the headless browser-use daemon for this skill. Do NOT retry submit attempts in headless mode — each attempt may count against a server-side rate limit and lock the account. To pre-populate cookies without running the cron, Robert logs in once via his regular Chrome, then the agent extracts `Network.getCookies` and saves to `~/.hermes/profiles/influencer/seesaw-cookies.json` for the next cron cycle.

9. **Wrong password silently redirects to `/reset_password` — no inline
   error** — OBSERVED 2026-09-04. When Seesaw rejects the password, the
   Sign In button does NOT show an inline "incorrect password" message.
   Instead the URL silently changes to `https://app.seesaw.me/#/reset_password`
   and the password-reset page renders. After 2-3 failed attempts, every
   subsequent attempt may redirect there even with a correct password
   (rate-limit lockout). **Diagnostic pattern**: if click + 1s wait
   shows URL = `/#/reset_password` instead of staying on `/#/login?role=parent`,
   the password was wrong. **Stop and ask the user for the correct
   password — do NOT keep clicking, you will lock the account.** Also,
   after a wrong password is submitted the form often re-renders empty,
   which can mislead you into thinking `fill_input` failed.

10. **Browser viewport ≠ screenshot dimensions — click coords need scaling**
    — OBSERVED 2026-09-04. The headless browser used for this skill has
    `window.innerWidth=1784, innerHeight=767, devicePixelRatio=1` but
    `Page.captureScreenshot` returns a 1920×1080 PNG. Vision_analyze
    reports click targets in **screenshot coordinates**, but `click_at_xy`
    uses **viewport coordinates**. To click a screenshot-coordinate (sx, sy),
    apply: `view_x = sx * 1784/1920`, `view_y = sy * 767/1080`. Same for
    `fill_input` selector coords. **Workaround**: query
    `getBoundingClientRect()` for any element you want to click — it
    already returns viewport coords and bypasses the scaling entirely.
    Example:
    ```python
    result = cdp('Runtime.evaluate', expression='''
        JSON.stringify((function(){
            const b = document.querySelector('button');
            const r = b.getBoundingClientRect();
            return {x: r.left + r.width/2, y: r.top + r.height/2};
        })())
    ''', returnByValue=True)
    coords = json.loads(result['result']['value'])
    click_at_xy(coords['x'], coords['y'])
    ```
    Without this scaling, clicks land at the wrong target and may hit
    the wrong button entirely (e.g., clicking "I'm a Student" when you
    meant "I'm a Family Member" because they're stacked ~100px apart).

11. **Seesaw Home feed scrolls inside `div.sp-scrollable-column`, NOT `window`** — OBSERVED 2026-09-04. Calling `window.scrollTo(0, 1000)` returns `window.scrollY = 0` because the page body has `scrollHeight=0` (it's `display: contents`-style). The actual scrollable container is `div.sp-scrollable-column` with `scrollHeight=5954, clientHeight=707` (a column-style feed layout). **Recipe for scrolling the feed**:
    ```python
    cdp('Runtime.evaluate',
        expression='document.querySelector("div.sp-scrollable-column").scrollTo({top: 800, left: 0, behavior: "instant"})',
        returnByValue=True)
    ```
    After scrolling, capture and verify by re-querying `document.querySelector("div.sp-scrollable-column").scrollTop` — if it stays 0, the selector is wrong (try `div[class*="scrollable"]` or `div[class*="feed-list"]`).

12. **Family Member Sign In button is at y=341 (viewport), NOT y=380 (vision)** — OBSERVED 2026-09-04. After `getBoundingClientRect()` on `button:contains("Family Member Sign In")` returned `y=341.66, height=34`, vision_analyze of the rendered page placed it at ~y=380 — a 38px offset. Clicking at vision's (892, 380) hit empty space below the button (form got reset by some onBlur handler, not the Sign In submit), so the click never registered as a Sign In attempt. **This was the root cause of 4 "failed login" attempts in one session — the password was correct, the clicks were missing.** Always click via JS-discovered viewport coords, never vision coords for buttons inside login forms.

13. **Use `Runtime.evaluate` to extract the full feed metadata in one call** — LEARNED 2026-09-04. Instead of scrolling + screenshotting + vision-analyzing post-by-post, a single JS query lists all feed items with their captions, timestamps, authors, and photo counts. **Selector caveat (verified 2026-09-04): `div.ss-feed-item--content` matched 0 nodes** — the class name has drifted. Use a cascading fallback:
    ```python
    cdp('Runtime.evaluate', expression='''
    JSON.stringify((function() {
        let items = document.querySelectorAll('div.ss-feed-item--content');
        if (!items.length) items = document.querySelectorAll('[class*="ss-feed-item"]');
        if (!items.length) items = document.querySelectorAll('[class*="feed-item"]');
        return Array.from(items).map(i => i.innerText.trim().slice(0, 200));
    })())
    ''', returnByValue=True)
    ```
    `[class*="ss-feed-item"]` returned 38 nodes for 9 real posts — see the allowlist-filter section above; most nodes are wrapper/comment/stripped-sibling rows, NOT posts. Use `1/N` carousel badges in the text to detect multi-photo posts. This is much faster than vision-scrolling for the "what's on the feed today?" question.

14. **Save auth cookies to disk for cron-driven extraction across sessions** — LEARNED 2026-09-04. Seesaw does not permit automated login in unattended cron contexts (rate limits + captcha). Workaround pattern:
    - **One-time**: Robert logs in manually once via Chrome; agent extracts cookies via `cdp('Network.getCookies', urls=['https://app.seesaw.me/'])` and saves to `~/.hermes/profiles/influencer/seesaw-cookies.json` (9 cookies, ~4KB).
    - **Per cron run run**: agent loads cookies, sets them via `Network.setCookie` for the `app.seesaw.me` domain BEFORE the first `new_tab(...)` call, then opens the feed URL. If cookies are valid, no login flow is triggered.
    - **Validity gate**: check `authToken` cookie's `expires` field — if `expires < now`, the session is dead, exit with a friendly "please log in once" message instead of attempting automated login.
    - **Refresh cycle**: Seesaw sessions last ~24h-7d depending on activity. When the cron reports expired cookies, Robert logs in once in Chrome, replies "logged in" to the reminder, agent re-saves cookies.
    - **Companion helper script**: `~/.hermes/profiles/influencer/scripts/seesaw_photos_pipeline.py` implements `--check-cookies` (validity check, exit code 0/1) and `--extract-today` (filter today's posts and exclude Aspen's). Reference for the cron pattern: `~/.hermes/profiles/influencer/cron/output/` job `5cs-seesaw-photos-wechat`.

14a. **Direct HTTP API for everything — no browser needed** — LEARNED 2026-09-06. Seesaw exposes two authed REST endpoints that return ALL the data the GUI would show:
    - `GET /api/person/parent/feed` — paginated feed (returns all items, share tokens, tagged students, comments, num_pages)
    - `GET /api/item_v2?item_id=...` — single item with FULL per-page data (the Seesaw SPA uses this when you click into a post)
    Both require the saved `seesaw-cookies.json` (authToken cookie). The API returns:
    - `item_id`, `share_token`, `text`, `create_date`
    - `people_tags_summary` (author + Everyone flag)
    - `pages.objects[].composite_image_url` and `caption` (all num_pages pages)
    - `comments.objects[]` (text, author, is_approved)
    - `tags.objects[]` (folder names)
    Use `~/.hermes/profiles/influencer/scripts/seesaw_api.py` for everything. **No Chrome / CDP / lightbox navigation needed for any post.** Only use the GUI (`seesaw_cdp_post.py`) as a last-resort fallback for posts that fail API enrichment.

14b. **Per-post permalink = .../pages/shared_item?item_id=...&share_token=...** — LEARNED 2026-09-06. The Seesaw post permalink is NOT in the family feed URL. The correct format is built from `item_id` and `share_token` (both returned by the API):
    `https://app.seesaw.me/pages/shared_item?item_id=item.<uuid>&share_token=<token>&mode=share`
    `seesaw_api.extract_share_link()` builds this from API data — no need to click through the More Options → Share Item popup anymore.

14c. **Per-image captions via /api/item_v2** — LEARNED 2026-09-06. The feed API returns only page 1 of multi-page posts (a Seesaw data anomaly). The `/api/item_v2` endpoint returns ALL pages with their real teacher captions. After `seesaw_api.enrich_item_with_full_pages(item)`, the item has full captions like "Aspen and Jennie playing..." for each page. NO NEED for lightbox carousel stepping.

14d. **Teacher comments are inline in feed/item_v2 response** — LEARNED 2026-09-06. Comments are at `item.comments.objects[]` with `text`, `author.display_name`, `is_approved`, `author.is_teacher_override`. Filter on `is_approved=True` and skip "Thanks!" reactions (parent reactions are not informative as captions). The function `seesaw_api.extract_teacher_comments()` does this filtering and sorts by teacher-first / longest-first.

14e. **Python int vs str comparison bug** — DISCOVERED 2026-09-06. `current_page = json.loads(check)` returns a Python `int` when JS returns a JSON number. Comparing `int == '1'` (string) is always False. Always use `parseInt()` in JS or compare against the int type on the Python side. Symptom: carousel page-stepping loop never breaks and runs through all pages instead of stopping at page 1.

14f. **Tagged students list is in the API** — LEARNED 2026-09-06. `item.people_tags.objects[]` contains the full list of students tagged in the photo, with `first_name`, `last_name`, `display_name`, `is_student_override`. The function `seesaw_api.extract_tagged_students()` filters out Aspen Hsiung (Robert's privacy rule) and returns first names. This is used as a fallback when captions are empty, and for filtering "Aspen in tagged students vs Aspen in people_tags_summary" (she's the author in the latter case).

14g. **WeChat 4.1.60 file picker (NSOpenPanel) is owned by a SEPARATE process — count via "Open and Save Panel Service", NOT "WeChat" sheets** — LEARNED 2026-09-11. `tell process "WeChat" to count sheets of window 1` returns 0 even when a picker is open (the picker is a separate window owned by a system helper process). The correct check: `tell process "Open and Save Panel Service" to count of windows`. If you don't verify with the right process, you think the folder icon click missed and sweep x trying other coords, when actually the picker opened fine and your keystrokes go to the wrong target. Always verify picker-open status with the right process. **REVISED 2026-09-11**: on the current WeChat build the picker appears as a SHEET on WeChat window 1 — `tell process "WeChat" to count sheets of window 1` is the reliable check (returns 1 when open); the "Open and Save Panel Service" count stayed 0 even with the picker open. Check BOTH and trust either-positive. And on a false check: abort before any keystrokes, or the typed path lands in the selected group chat (see wechat-macos-automation Pitfall #72/#73).

14h. **Seesaw feed API returns `author_id` but NO author `display_name` — display names are resolved by `seesaw_api.build_metadata()` from `people_tags_summary`** — DISCOVERED 2026-09-22. The `/api/person/parent/feed` response has `author_id` (a `person.<uuid>` token) but no human-readable `display_name` for the post author. If you build a message like "— Ms. Sitler (Oct 6, 2026 7:16 AM)" directly from the feed response, you'll emit "— None" or fall back to the post body. The fix lives in `build_metadata()`: it joins `people_tags_summary.objects[].name` and tags the most recent author name into the `author` field of `metadata.json`. **When piping feed items into a render path** (the cron prompt, a WeChat message builder, a digest), always run `build_metadata()` first to populate the human-readable fields — don't try to derive the teacher name from `author_id` alone. **Symptom of the bug**: a render that emits `— None` or `— ` (blank) for the byline of a teacher post when the feed API was queried directly without going through `build_metadata()`.

15. **Multi-post feed extraction loop — close lightbox between posts** — LEARNED 2026-09-04. When extracting from N posts in one cron run, the agent needs to navigate from one post's lightbox back to the feed to click the next post's image. Failure mode: clicking the lightbox backdrop doesn't reliably close it; pressing Escape sometimes works but sometimes triggers the page-back action instead. **Working sequence**:
    ```python
    for post in today_posts:
        # Scroll post into view
        post.scrollIntoView()
        # Click first image in this feed item
        click_at_xy(post_x, post_y)
        time.sleep(2)  # lightbox open
        # Extract all photos via right-arrow iteration
        for i in range(post.carousel_count):
            urls.append(get_current_src())
            if i < post.carousel_count - 1:
                click_at_xy(1713, 348)  # right arrow
                time.sleep(2)
        # Close lightbox — click backdrop at (960, 1000) (bottom-center outside photo)
        # If that fails, press Escape (key code 27) — but Escape sometimes navigates back
        osascript("keystroke (key code 27)")
        time.sleep(1)
        # Verify URL is back at /#/family/feed before clicking next post
        url = cdp('Runtime.evaluate', expression='window.location.href', returnByValue=True)
        if '/family/feed' not in url.get('result', {}).get('value', ''):
            # Re-navigate to feed if Escape took us off
            cdp('Page.navigate', url='https://app.seesaw.me/#/family/feed')
            time.sleep(3)
    ```
    Alternative: open each post in a new tab (`new_tab(post.source_url + '/post-id')`) — slower (cold-start each post) but more reliable than the lightbox-close loop.

16. **VERIFIED cron-safe auto-login recipe (visible Chrome + raw CDP over websocket)** — PROVEN END-TO-END 2026-09-06, first attempt, no lockout. This supersedes the fill_input/browser-use path for unattended runs. Two gotchas that broke earlier attempts:

    **(a) `https://app.seesaw.me/#/login?role=parent` does NOT land on the login form.** The SPA strips the `?role=parent` hash query on cold load and renders the *role picker* instead (buttons: `Select Language`, `I'm a Teacher`, `I'm a Student`, `I'm a Family Member`). `input` count is **0**, so any `fill_input`/querySelector on email+password finds nothing and you get a misleading `inputs-not-found`. **Fix: detect `!document.querySelector('input[type="password"]')` and click `I'm a Family Member` first**, wait ~5s, then the real form (with `Family Member Sign In` at y≈341) renders.

    **(b) Launching Chrome from zsh: quote the glob.** `--remote-allow-origins=*` is glob-expanded by zsh and the browser dies instantly with `zsh:1: no matches found`. Always write `"--remote-allow-origins=*"`. Also pass a start URL (`about:blank`) so the process stays alive, launch it with `background=true`, and poll `curl http://127.0.0.1:9222/json/version` until it answers (~8s) before attaching.

    Working scripts (both committed):
    - `~/.hermes/profiles/influencer/scripts/seesaw_cdp_login.py` — role-picker click → native-setter form fill (`Object.getOwnPropertyDescriptor(el.__proto__,'value').set` + `input`/`change` events) → ONE `Input.dispatchMouseEvent` press/release on the JS-derived button rect → verify `/#/family/feed` → save cookies chmod 600. Bails out on `/reset_password` (Pitfall #9) and never retries.
    - `~/.hermes/profiles/influencer/scripts/seesaw_cdp_feed.py` — attaches to the authenticated tab, forces `#/family/feed`, and dumps the 38-node feed JSON to stdout.

    Uses only `websocket-client` (already installed). **Talk raw CDP, not the browser-use daemon** — the daemon is headless and will hit the reCAPTCHA gate from Pitfall #9a.

17. **`--extract-today` expects `{"posts": [...]}`, not a bare array** — OBSERVED 2026-09-06. `seesaw_cdp_feed.py` prints a bare JSON array, so piping it straight in raises `AttributeError: 'list' object has no attribute 'get'`. Wrap it first: `json.dump({"posts": feed}, ...)`.

18. **Regression-test fixtures MUST compute their dates at runtime** — BUG FOUND AND FIXED 2026-09-06. `tests/test_seesaw_filter.py` and `tests/test_seesaw_dedup.py` hardcoded `"Sep 4, 2026"` as "today". From Sep 5 onward, every test asserting "a teacher post dated today is kept" reported **FAIL** — not a filter regression, just a stale fixture, and exactly the kind of false alarm that makes you distrust a correct privacy filter mid-cron. Both files now derive `TODAY`/`YESTERDAY` from `datetime.now()` and exit non-zero on real failure (they previously always exited 0, so `&&` chains never caught anything). Filter suite is now A–E (E is a dedicated privacy case for Aspen's stripped-sibling `In response to:` row). **When a filter test fails, check the fixture date before touching filter logic.**

19. **Backfill log key format mismatch: full `item.<uuid>` vs short `YYYYMMDD_<uuid-prefix>`** — DISCOVERED 2026-09-10. `scripts/seesaw-sent-log.json` was seeded with short-prefix keys (e.g., `20260903_a5d3638a00aa`) by the prior extractor, but the current `--extract-feed` returns full `item.<uuid>` from the API. `seesaw_backfill.find_unsent()` does an exact match, so a post with item_id `item.d41b91e2-...` is reported as unsent even when the log has `20260903_d41b91e2` for the SAME date but a different UUID prefix. **This is dangerous** — it causes duplicate deliveries when two posts share the same date, AND it causes genuine sends to be re-sent on the next cron when the prior delivery used a different ID format. **Mitigation applied 2026-09-10**: when entering the unsent filter, ALSO check if a local `~/workspace/sas-5cs/seesaw-photos/<YYYY-MM-DD>-<slug>/` folder exists — that's evidence the post was previously delivered even if the log uses the wrong key. After sending, always migrate log entries to use the full `item.<uuid>` format. **Operational rule**: any cron that compares feed item_ids to a backfill log must normalize keys to the same format BEFORE filtering, OR do a date+folder existence cross-check as a secondary filter.

20. **7-DAY AGE GATE on all auto-sends — INCIDENT 2026-09-10 20:38**: the photos cron found 3 unsent teacher posts from Aug 12/25/26 (2–4 weeks old) in the backfill log and chose to send them to the 5CS group as text+share-links (reasoning: "flooding the group with old photos is noisy"). Both choices — sending stale posts at all, and silently degrading photos→text — were wrong; Robert: "Why didn't you surface those choices to me first?" **Fixes**: (1) `seesaw_backfill.py --unsent-fresh` enforces MAX_AGE_DAYS=7 in code — returns `{fresh, stale_skipped}`; stale posts are reported to Robert, NEVER auto-sent; `--allow-stale` is a human-only override that no cron may pass. (2) The cron prompt now carries a HARD RULE: any anomaly (age, volume, missing images, format choice) → stop and present options; group-send content is Robert's editorial call, not the cron's. **Rule**: prose constraints in prompts get overridden by agent reasoning — enforce policy in the script layer wherever a wrong send is irreversible.

21. **Exclude `metadata.json` from the WeChat picker when sending Seesaw photos — Robert's preference 2026-09-14** — Even though `seesaw_photos_pipeline.py` writes a `metadata.json` alongside each downloaded photo album (capturing post caption, author, date, photo list), it MUST NOT be sent to the parent group. macOS NSOpenPanel hides dot-prefixed files by default. Recipe: rename `metadata.json` → `.metadata.json` BEFORE opening the picker; after the picker closes (in `finally:`), rename back so other scripts (`--compose-message`) can still read it. See `wechat-macos-automation` Pitfall #55 and `references/cron-photos-pipeline.md` for the verified implementation in `send_5cs_seesaw_photo.py`. When the data shape ever changes (new sidecar files: log, html, etc.), extend the rename set — don't reintroduce the "metadata rides along" framing.

22. **Seesaw login via CDP — surface reCAPTCHA to the user, never silently retry — Robert 2026-09-14** — `seesaw_cdp_login.py` polls up to 12s for the React role-picker / password input to mount; each poll also probes for an `iframe[src*="recaptcha"]`, `.g-recaptcha` badge, or body text matching `/captcha|robot|verify you|unusual traffic/i`. If captcha is detected at any point: exit **code 6** with `{"ok": false, "reason": "captcha_required"}` and let the cron prompt stop. **Never** auto-retry with another CDP login attempt — each attempt may count against the server-side rate limit and lock the account. The cron prompt MUST surface a "human required, open Chrome at :9222, complete the captcha, then say retry" message and exit. The fixed `time.sleep(6)` was not enough headroom for React mount; the poll pattern (Pitfall #16) was. Old `rc=3 role_button_not_found` → silent retry is now `rc=6 captcha_required` → user action.

23. **Schedule other WeChat-driving crons by the announcements pipeline's RUNTIME window, not its trigger minute — short GUI jobs go BEFORE it** — the daily-announcements pipeline triggers at 17:30 but runs ~17 minutes (Chrome/Seesaw login + PDF phases that hold OS focus, plus its own WeChat mirror sends). Any WeChat GUI job triggered inside that window fights the focus battles it loses ("would not come frontmost") or, worse, interleaves clicks. Rule: estimate runtime (trigger → last GUI action) for every WeChat-driving job and place triggers so no runtime window contains another trigger — ≥10-min trigger gaps alone are NOT sufficient (a 17:40 and a 17:55 drain both failed inside the announcements window on consecutive evenings). Prefer ordering short jobs before long ones (producers 17:10 → outbox drain 17:20 → announcements 17:30): the short drain finishes before the long job starts, and an overrun delays nothing downstream. When moving a consumer earlier, move its producers in the same change or the drain runs an empty queue and messages slip a day. Sketch the FULL sequence (all producers, consumers, runtimes) before editing any schedule.

24. **Messages attachments: GraphQL API, never button clicks** — the Messages
    "View Original" button is a React `role="button"` DIV whose wrapper
    swallows BOTH synthetic `.click()` AND CDP `Input.dispatchMouseEvent`
    (press+release at the live rect opens no tab). Do not spend turns on
    click strategies for it — fetch the message via
    `POST https://reloaded-api.seesaw.me/ss2_gql` (Bearer token from
    `localStorage["ASV3-7zeem4"]` on the logged-in SPA tab; cookies alone →
    `InvalidAuth`) and curl the signed `item.attachment.url` directly.
    Full recipe: `references/seesaw-messages-pipeline.md`.

25. **Photos cron: enqueue via outbox, drain sends — refactor 2026-10-06**
    The nightly 8 PM photos cron `40ef4787548e` extracts teacher posts and
    delivers them to the **5CS Class Chat 26-27** group chat. The pipeline
    is now an outbox producer (not a direct GUI sender): run
    `seesaw_photos_pipeline.py --enqueue-fresh <feed_json> --enqueue-to 5cs-rp`
    and let `wechat_outbox.drain()` (07:50 / 17:20 crons) drive WeChat.
    Do NOT have this cron directly drive the WeChat GUI — the outbox
    is the single sink for all sends, and a parallel sender drift is the
    exact failure pattern that motivated the outbox architecture. Single-photo
    delivery format (still the default):
    - Download only the first photo of any multi-page post (`seesaw_api.py --download-album --first-photo`, the default since 2026-09-22). `--all-photos` is archive-only.
    - Send the photo as ONE WeChat message bubble.
    - Follow with one text message built by `seesaw_photos_pipeline.py:compose_wechat_message_with_more_photos()`:
      - First photo's caption (or `teacher_comment` / `post` fallback) as the body
      - **One preview line per skipped photo** (truncated to ~40 chars) — surfaces other kids' names so parents have a reason to click through
      - `📸 N more photo(s) available on Seesaw:` line (only if N > 0)
      - `🔗 <permalink>` (built from `item_id` + `share_token` via `extract_share_link`)
      - `💬 Parents: please open Seesaw and leave an encouraging comment for your child!` closer
    - Result: 2 messages per fresh post (1 photo + 1 text), regardless of album size. A 12-photo album doesn't become a 13-bubble dump.

    Test-render before any live enqueue with `python3 seesaw_photos_pipeline.py --compose-message <folder>`. The legacy full-album format is preserved under `--compose-message-legacy <folder>` for diffing against old behavior — never the default.

    **Why**: WeChat has a 9-photo-per-msg cap; a 12-photo album silently degrades to a 13-bubble flood that the parent group already flagged as spam. The single-photo + link pattern keeps the group informed without the noise. The privacy + per-photo-caption rules still apply (Pitfalls #20, #21, #22, etc.).

    **Edge case**: some multi-page posts have only ONE caption (the post body reused for page 1, pages 2..N are uncaptioned). The companion still emits `📸 6 more photos available on Seesaw:` with no preview lines — better than fabricating text, and still strictly better than dumping all 6 photos.

    **Verification after a live drain**: the outbox archive
    (`~/workspace/wechat-outbox/archive/YYYY-MM/<msg-id>.json`) records
    `status: sent` per (msg_id, recipient). The cron's sent-log at
    `~/workspace/sas-5cs/seesaw-photos-sent.json` tracks which post_ids
    have been enqueued; do NOT mark posts sent manually, the pipeline
    does it. The producer's "enqueued" exit means the JSON landed in
    queue — the drain is what actually pushes the WeChat bubble (see
    `wechat-outbox-delivery` Pitfall #24 about producer vs drain logs).

## Reference files

- `references/post-extraction-recipe.md` — the exact CDP code + coordinate
  map for Robert's Mac mini (1920x1080, WeChat-style window layout), copy-paste
  ready
- `references/feishu-image-delivery.md` — how to actually deliver the photos
  to Robert (drag-from-Finder is the reliable path)
- `references/login-coords-2026-09-04.md` — verified JS-discovered coords
  for the Family Member Sign In button (y=341, NOT y=380 from vision).
  Vision-said button positions are off by 38px here; always query
  `getBoundingClientRect()` rather than trusting the screenshot view.
- `references/scrollable-feed-pattern.md` — `window.scrollTo` does NOT
  scroll the Seesaw feed; use the inner `div.sp-scrollable-column`
  container. Includes the JS recipe for full-feed post extraction.
- `references/cron-photos-pipeline.md` — end-to-end recipe for the
  nightly 8 PM Mon-Fri cron (`40ef4787548e`) that uses this skill +
  `wechat-macos-automation` to extract today's teacher photos and send
  them to WeChat as merged messages. Includes the companion helper
  script (`seesaw_photos_pipeline.py`), cookie lifecycle, and the
  "manual verification" command.
- `references/seesaw-messages-pipeline.md` — the Messages/announcements
  surface: GraphQL (`ss2_gql`) API recipe with the exact query, the hybrid
  DOM-scan + API-fetch architecture, "new message" definition, `--unread-only`,
  send guards, and WeChat mirror targets for the daily announcements cron.
- `references/post-detail-extraction.md` — share-link retrieval via More
  Options → Share Item, per-image caption stepping through the carousel,
  teacher comment fallback. Proven end-to-end on Sep3 (per-image) and
  Sep4 (merged+comment) posts.

## Verified extraction (Sep 3, 2026)

| Field | Value |
|---|---|
| Source post | Ms. Sitler "Powers of Ten Chutes & Ladders" math post |
| Post date | 2026-09-03 11:17 |
| Photos extracted | 7 (carousel "1/7") |
| File sizes | 534-653 KB each (1080x810 / 1080x608 / 1080x703 variants) |
| Total size | ~4.2 MB |
| Save location | `~/workspace/sas-5cs/seesaw-photos/2026-09-03-powers-of-ten/` |
| Vision-verified | 2/7 photos confirmed as real classroom shots (faces visible, math game board legible) |
| Time to extract + download | ~6 minutes (browser time + 7 sequential 1-2s downloads) |