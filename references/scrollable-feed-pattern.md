# Seesaw Home Feed Scroll + Extraction (verified 2026-09-04)

## Why this file exists

The Seesaw Home feed does NOT respond to `window.scrollTo()`. The page body
has `scrollHeight=0` (the layout uses `display: contents` or similar), so
the actual scrollable element is a child div. The agent initially tried
multiple `window.scrollTo(0, 1000)` calls, all of which silently did nothing,
and the screenshot didn't update.

## The actual scrollable container

```
div.sp-scrollable-column  ← scrollHeight=5954, clientHeight=707
```

To find the scrollable container generically:

```python
result = cdp('Runtime.evaluate', expression='''
JSON.stringify(Array.from(document.querySelectorAll('*')).filter(e => {
    const s = getComputedStyle(e);
    return (s.overflowY === 'auto' || s.overflowY === 'scroll') &&
           e.scrollHeight > e.clientHeight;
}).slice(0, 5).map(e => ({
    tag: e.tagName,
    cls: e.className?.slice(0, 80),
    sh: e.scrollHeight,
    ch: e.clientHeight
})))
''', returnByValue=True)
```

## Recipe: scroll the Seesaw feed

```python
# Scroll to absolute position
cdp('Runtime.evaluate',
    expression='document.querySelector("div.sp-scrollable-column").scrollTo({top: 800, left: 0, behavior: "instant"})',
    returnByValue=True)
import time; time.sleep(2)

# Verify scroll worked
result = cdp('Runtime.evaluate',
    expression='document.querySelector("div.sp-scrollable-column").scrollTop',
    returnByValue=True)
print(f"Container scrollTop: {result['result']['value']}")  # should be 800
```

## Recipe: dump the entire feed metadata (no scrolling required!)

The fastest way to enumerate all visible posts is a single JS query:

```python
result = cdp('Runtime.evaluate', expression='''
JSON.stringify((function() {
    const items = document.querySelectorAll('.ss-feed-item--content');
    return Array.from(items).map(i => {
        const text = i.innerText.trim();
        // Extract photo count from "1/N" badges if present
        const carouselMatch = text.match(/(\d+) \\/ (\\d+)/);
        return {
            text: text.slice(0, 300),
            photos: carouselMatch ? parseInt(carouselMatch[2]) : 1,
            hasCarousel: !!carouselMatch
        };
    });
})())
''', returnByValue=True)
```

Returns ~20 posts with their captions, authors, timestamps, and photo
counts. Each item's `.text` contains:
- Author (e.g., "Ms. Sitler", "Aspen Hsiung", "Lindel Limpisvasti")
- Date/time (e.g., "Sep 4, 2026 9:25 AM")
- Carousel indicator (e.g., "1 / 7") — only present for multi-photo posts
- Caption (the post body)
- Tagged folders (e.g., "5CS - 26-27 SY", "Class Photos 📷")

## Real feed dump example (2026-09-04 09:55 CST)

```
Pinned: Ms. Sitler "5CS Specials Rotation Schedule" — Aug 12
Aspen Hsiung "CLL4 U1 (预测) ETK默写" — Sep 4 09:25 AM, 1 photo
Ms. Sitler "Max and Albert playing a 'Powers of Ten Chutes & Ladders'..." — Sep 3 11:17 AM, 7 photos
Ms. Sitler "Happy Birthday, William!" — Aug 31 3:11 PM, 1 photo
Ms. Sitler "5CS students went to the iHub..." — Aug 26 12:35 PM, 1 photo
Lindel Limpisvasti "LOOK IN: Fitness and Locomotor Unit..." — Aug 25 3:07 PM, 20 photos
Ms. Sitler "Sidewalk & Sprinklers 2026!" — Aug 21 2:47 PM, 11 photos
Aspen Hsiung (response: Seesaw Tools: Lessons 6-10) — Aug 21 11:03 AM, 7 photos
Ms. Sitler "Happy Birthday, Leon!" — Aug 19 12:10 PM, 1 photo
```

Useful for answering "what's on the feed today?" without scrolling.

## When to scroll vs. when to dump

- **"Is there anything from today?"** → dump, search for today's date in text
- **"Download photos from the X post"** → scroll to that post, click the image,
  use the lightbox extraction recipe (`post-extraction-recipe.md`)
- **"Archive everything visible"** → dump to enumerate, then loop with lightbox
  extraction per multi-photo post

## Selector stability warning

`.ss-feed-item--content` is a Seesaw-specific class observed on 2026-09-04.
Seesaw occasionally renames classes with framework updates — if the dump
returns empty, inspect the DOM and look for the new feed-item container
class.