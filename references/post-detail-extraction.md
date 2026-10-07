# Post Detail Extraction (share link, per-image captions, teacher comments)

PROVEN END-TO-END 2026-09-06 on the Sep 3 Powers of Ten post (7 photos) and the
Sep 4 Signify exhibition post (4 photos, no per-image captions).

## What this extracts

For a Seesaw post already open in the visible Chrome on `:9222`:

- **`share_link`** — the proper per-post permalink
- **`embed_code`** — iframe embed code (fallback)
- **`image_captions`** — list of `{page, caption}` for each photo (one entry per carousel page)
- **`comments.teacherComments`** — teacher comments on the post (used as caption for merged albums)

## Helper script

`~/.hermes/profiles/influencer/scripts/seesaw_cdp_post.py`

Run modes:

```bash
# Full extraction in one shot
python3 seesaw_cdp_post.py --caption-hint "Powers of Ten" --full

# Just the share link
python3 seesaw_cdp_post.py --caption-hint "Powers of Ten" --extract-share-link

# Just per-image captions
python3 seesaw_cdp_post.py --extract-image-captions

# Just teacher comments
python3 seesaw_cdp_post.py --extract-comments
```

`--caption-hint` is a substring of the post caption used to pick the right
post in the feed (the first match wins). Without it, the script picks the
post nearest to `--post-x`/`--post-y` viewport coords.

## Share link extraction — step by step

1. **Click "More Options"** — `[aria-label="More Options"]` (MuiIconButton).
   Direct DOM click works; CDP mouse click sometimes misses. Use:
   ```js
   document.querySelector('[aria-label="More Options"]').click();
   ```

2. **Click "Share Item"** — last item in the popup menu, role="menuitem":
   ```js
   const items = Array.from(document.querySelectorAll('[role="menuitem"]'));
   items.find(i => i.textContent.trim() === 'Share Item').click();
   ```

3. **Read the share link** — the popup exposes two `<input type="text">`:
   - `values[0]` = share link (`https://app.seesaw.me/pages/shared_item?item_id=item.<uuid>&share_token=<token>&mode=share`)
   - `values[1]` = embed code (`<iframe height='400' width='400' src='...'></iframe>`)

4. **Close the popup** with Escape before continuing (other queries may fail
   if the modal context is wrong).

## Per-image caption extraction — step by step

The Sep 3 Powers of Ten post has a caption on EACH photo (each names different
students in that photo). The captions live in a single DOM element that
updates as the carousel advances:

```css
.css-901oao.r-11sbt46.r-1adg3ll.r-1vywy00.r-ubezar.r-s1qlax
```

Inside `.sp-item-modal` (scoped to the lightbox — do NOT query `document.body`
because background posts also have feed-item text).

To extract all captions:

1. Click into the post to open the lightbox (the post caption is shown as
   the lightbox title).
2. **Step back to page 1** — click "previous page" until counter shows `1 / N`.
   The carousel is CIRCULAR (clicking prev at page 1 loops to page N), so
   check the counter and break when it reads `1`.
3. For each page, read the caption text via the selector above.
4. Click "next page" (`[aria-label="next page"]`) to advance. Repeat.

**Critical bug to avoid**: the page counter from `document.body.innerText`
returns multiple matches (feed has counters too — `1 / N` for each post's
carousel preview). Always read the counter from `modal.innerText`:

```js
const modal = document.querySelector('.sp-item-modal');
const counter = modal.innerText.match(/(\d+)\s*\/\s*(\d+)/);
// counter[1] = current page, counter[2] = total pages
```

## Teacher comment extraction

Some posts (like the Sep 4 Signify exhibition) have NO per-image captions but
DO have a teacher comment explaining the post. The comment is inside
`.sp-item-modal` with `class*="comment"`.

To filter to teacher comments only, match against known teacher names:

```js
const teacherNames = ['Ms. Sitler', 'Mr. Limpisvasti', 'Ms. Lily', 'Ms. Sato',
                     'Mr. Roselle', 'Ms. Ellie Brown', 'Ms. Beard', 'Ms. Kiran',
                     'Ms. Victoria', 'Ms. Lindel', 'Mr. Sean', 'Mr. Briers',
                     'Ms. Wallace', 'Ms. Sue', 'Mr. Q', 'Mr. Hargreaves'];
```

The raw comment text has Seesaw UI annotations that must be stripped before
sending:

```python
import re
text = re.sub(r'\s*\(from blog\)', '', raw)
text = re.sub(r'\s*\(Pending Approval\)', '', text)
text = re.sub(r'Ms\.?\s*\w+\s*', '', text, count=1)  # strip author name prefix
text = re.sub(r'Mr\.?\s*\w+\s*', '', text, count=1)
text = re.sub(r'\s+', ' ', text).strip()
```

## Python int vs str comparison bug

When the JS does `parseInt(counter[1])` and you do:

```python
current_page = json.loads(check)  # returns int, not str!
if current_page == '1':  # always False — int != str
    break
```

The comparison fails silently and the carousel-page-stepping loop never
breaks — it runs through all pages instead of stopping at page 1.

**Fix**: either compare against the int (`if current_page == 1`) or wrap
the JS return in `String()` (e.g., `String(parseInt(...))`) before
`json.loads` so Python sees it as a string.

## Send strategies (caller decision)

After extraction, decide based on `image_captions`:

- **Per-image captions present** (at least one entry with non-empty caption):
  send each photo individually as a separate WeChat message with that
  photo's caption + the permalink + the encouraging note.
- **No per-image captions** (all captions empty): merge all photos into ONE
  WeChat bubble using the "Show as merged photos" feature (see
  wechat-macos-automation Pitfall #50, merge bubble at coords (363, 960)),
  then one text message using the teacher's comment as the caption + the
  permalink + the encouraging note.

## Verified examples (Sep 3, 2026)

```json
{
  "post": {"caption": "1 / 7\nMax and Albert playing a 'Powers of Ten'..."},
  "share_link": "https://app.seesaw.me/pages/shared_item?item_id=item.d41b91e2-0f40-4333-9cb9-5d619b141e58&share_token=K6H_-gakTkOw5suBsaneKQ&mode=share",
  "image_captions": [
    {"page": "1 / 7", "caption": "Max and Albert playing..."},
    {"page": "2 / 7", "caption": "Aspen and Jennie playing..."},
    ...
    {"page": "7 / 7", "caption": "Evelyn and Irene playing..."}
  ],
  "comments": {"teacherCommentCount": 0, "teacherComments": []}
}
```

For the Sep 4 Signify post (no per-image captions, has teacher comment):

```json
{
  "image_captions": [
    {"page": "1 / 4", "caption": ""},
    {"page": "2 / 4", "caption": ""},
    {"page": "3 / 4", "caption": ""},
    {"page": "4 / 4", "caption": ""}
  ],
  "comments": {
    "teacherCommentCount": 1,
    "teacherComments": [
      {"author": "Ms. Sitler", "text": "Ms. Sitler Signify (formerly Philips Lighting) brought their stunning Light, Nature & Sustainability exhibition — fresh from the Shanghai Natural History Museum — right to our PAC Lobby!"}
    ]
  }
}
```

After cleaning (`_clean_teacher_comment()` in `seesaw_photos_pipeline.py`):

```
Signify (formerly Philips Lighting) brought their stunning Light, Nature & Sustainability exhibition — fresh from the Shanghai Natural History Museum — right to our PAC Lobby!
```