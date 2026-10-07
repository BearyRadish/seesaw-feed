# Seesaw Post Extraction Recipe (CDP + clicks)

Verified copy-paste recipe for extracting all photo URLs from a Seesaw lightbox
post on Robert's Mac mini (1920x1080 logical). Adjusted for the "uiautomation"
class of browser harness available in Hermes.

## Setup

```python
# In browser_exec code blocks:
import json
import time
from pathlib import Path

# Helpers available:
#   new_tab(url)     — navigate (first time)
#   goto_url(url)    — navigate current tab
#   wait_for_load()
#   capture_screenshot() — returns path string, also auto-attaches image
#   js(expr)         — evaluate JS, BUT this session returned {} — UNRELIABLE
#   cdp(method, **kwargs) — raw CDP. THIS WORKS.
#   click_at_xy(x, y) — click viewport coordinates
#   fill_input(selector, text) — type into input
#   d('Domain.method', **kwargs) — alternative CDP form
```

## Find all clickable arrows in the lightbox

After clicking the post image to open the lightbox, query all elements that
#look like navigation arrows:

```python
res = cdp('Runtime.evaluate', expression="""
JSON.stringify((function() {
    const out = [];
    document.querySelectorAll('button, [role=button], [class*=arrow], [class*=next], [class*=prev], [class*=Arrow], [class*=Next]')
        .forEach(el => {
            const r = el.getBoundingClientRect();
            const t = (el.innerText || el.ariaLabel || el.title || '').slice(0, 30);
            if (r.width > 0 && r.height > 0) {
                out.push({
                    tag: el.tagName,
                    class: el.className.toString().slice(0, 80),
                    x: Math.round(r.left + r.width/2),
                    y: Math.round(r.top + r.height/2),
                    text: t,
                    w: Math.round(r.width),
                    h: Math.round(r.height)
                });
            }
        });
    return out;
})())
""", returnByValue=True)
arrows = json.loads(res['result']['value'])
# Filter to right-side arrows (x > 1000)
right_arrows = [a for a in arrows if a['x'] > 1000]
for a in right_arrows:
    print(f"  {a['tag']} @ ({a['x']}, {a['y']}) text='{a['text']}' class='{a['class'][:40]}'")
```

The arrow you want has `text='next page'` and the Seesaw CSS class signature
`css-18t94o4 css-1dbjc4n r-1habvwh r-tn79kl r-13c4fz4 r-1rkxksu r-fnigne r-12r24e`.
At 1920x1080 with the Seesaw default feed layout, it's typically at **(1713, 348)**.
The matching LEFT arrow is at **(191, 348)**.

## Iterate through the carousel

```python
def get_current_image_url():
    """Return the src of the currently-displayed lightbox image, or 'none'."""
    res = cdp('Runtime.evaluate', expression="""
    JSON.stringify((function() {
        const largeImgs = Array.from(document.querySelectorAll('img'))
            .filter(i => i.naturalWidth >= 1500);
        return largeImgs.length ? largeImgs[0].src : 'none';
    })())
    """, returnByValue=True)
    return res['result'].get('value', 'none')

def strip_url_quotes(url):
    """CDP JSON.stringify() wraps strings in literal double quotes."""
    return url.strip().strip('"')

def extract_n_photos(n):
    """Click right arrow n-1 times, collecting each photo URL in order."""
    urls = []
    for i in range(n):
        raw = get_current_image_url()
        urls.append(strip_url_quotes(raw))
        if i < n - 1:
            click_at_xy(1713, 348)   # right arrow in lightbox
            time.sleep(2)             # wait for next photo to load
    return urls

# Usage
urls = extract_n_photos(7)  # for "1/7" carousel
```

## Download all photos

```python
import urllib.request

def download_photo(url, out_path):
    req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
    with urllib.request.urlopen(req, timeout=20) as resp:
        out_path.write_bytes(resp.read())
    return out_path.stat().st_size

def download_all(urls, post_slug):
    out_dir = Path.home() / f"workspace/sas-5cs/seesaw-photos/{post_slug}"
    out_dir.mkdir(parents=True, exist_ok=True)
    results = []
    for i, url in enumerate(urls, 1):
        # URL pattern: https://assets.seesaw.me/.../<uuid>.jpg:::TS:::W:::#:::<TOKEN>.jpg
        # The UUID is the segment between / and :::
        uuid = url.split('/')[-1].split(':::')[0]
        out_path = out_dir / f"photo-{i:02d}-{uuid}.jpg"
        size = download_photo(url, out_path)
        results.append({'n': i, 'uuid': uuid, 'path': str(out_path), 'bytes': size})
        print(f"✓ photo-{i:02d} ({size//1024}KB)")
    return results
```

## Verify the batch

```python
import subprocess

def verify_photos(results):
    """Spot-check: file size, dimensions, vision_analyze on 1-2 random samples."""
    for r in results[:2]:
        # dimensions
        out = subprocess.run(['sips', '-g', 'pixelWidth', '-g', 'pixelHeight', r['path']],
                            capture_output=True, text=True)
        print(f"  {r['path']}: {out.stdout.strip().split(chr(10))[-2:]}")
        # vision_analyze outside this script
```

## Why these numbers

- **`naturalWidth >= 1500` for lightbox filter** — the lightbox image renders at
  1662x1247 actual pixels. Smaller images in the DOM are sidebar/feed thumbnails.
- **`time.sleep(2)` between clicks** — Seesaw lazy-loads the next photo into
  the DOM. 2s is enough; 1s sometimes missed, 3s is wasteful.
- **`(1713, 348)` right arrow** — derived from actual lightbox state, not
  assumed. The arrow is an inner DIV, not the screen-edge chevron.
- **`User-Agent: Mozilla/5.0`** — assets.seesaw.me returns proper JPEGs with
  this header; some other UAs get a CDN-block page.