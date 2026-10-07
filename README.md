# seesaw-feed

Programmatic access to the Seesaw **journal feed** (the family photo stream
where teachers post classroom photos and captions) via Seesaw's authed REST
endpoints.

Companion to [`seesaw-messages`](https://github.com/BearyRadish/seesaw-messages).
Seesaw exposes two completely separate APIs behind one login — the journal
feed is REST, Messages is GraphQL. This repo covers the feed.

## What you get

- Bulk-extract today's teacher posts without clicking into a lightbox.
- Full per-page photo URLs + captions (the feed endpoint only returns page 1;
  `/api/item_v2` returns all pages).
- Build a public `shared_item` permalink from `item_id` + `share_token`
  without navigating to the Share Item popup.
- Cron-safe: no reCAPTCHA, no Chrome required, ~100ms per post.

## Auth

Send the saved `authToken` cookie against `app.seesaw.me`. The cookie file
comes from a one-time Chrome `Network.getCookies` export and lasts ~24h-7d.
There is no headless-friendly login path — the `authToken` cookie expires
and must be refreshed by logging into Chrome interactively.

## Quickstart

```python
import json, urllib.request, os

cookies = json.load(open(os.path.expanduser('~/.hermes/seesaw-cookies.json')))
cookie_str = '; '.join(f"{c['name']}={c['value']}" for c in cookies)

req = urllib.request.Request(
    'https://app.seesaw.me/api/person/parent/feed',
    headers={'Cookie': cookie_str, 'Accept': 'application/json',
             'X-Requested-With': 'XMLHttpRequest',
             'User-Agent': 'Mozilla/5.0'})
data = json.loads(urllib.request.urlopen(req, timeout=15).read())
```

Then for any multi-page post, follow up with `GET /api/item_v2?item_id=...`
to get every page's caption and signed photo URL.

## Reference

- `SKILL.md` — full REST deep-dive: both endpoints, response shapes, key
  fields, pagination, the GUI fallback path (with coordinate map and CDP
  recipes), 25 verified pitfalls (reCAPTCHA, headless auth, lightbox
  navigation, privacy filtering, scheduling).
- `references/` — supplementary recipes:
  - `cron-photos-pipeline.md` — end-to-end nightly cron recipe
  - `post-extraction-recipe.md` — exact CDP code + coordinate map
  - `wechat-send-sequence.md` — delivery path
  - `caption-delivery-policy.md`, `feishu-image-delivery.md`,
    `login-coords-2026-09-04.md`, `scrollable-feed-pattern.md`,
    `post-detail-extraction.md`, `seesaw-messages-pipeline.md`,
    `session-alive-diagnostic.md`

## Environment variables

Set before running — never commit:

```bash
export SEESAW_EMAIL="<your-family-member-email>"   # for the one-time interactive login
```

The `authToken` cookie lives in a local cookies JSON file that this repo
gitignores.

## See also

- [`seesaw-messages`](https://github.com/BearyRadish/seesaw-messages) —
  companion repo for the Messages inbox GraphQL API
  (`reloaded-api.seesaw.me/ss2_gql`).
