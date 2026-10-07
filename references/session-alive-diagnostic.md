# Session-alive diagnostic for Seesaw (added 2026-09-04)

`cdp('Network.getAllCookies')` returns **zero cookies for `seesaw.me` even
when the session is fully authenticated**. Observed on 2026-09-04 in the
interactive session that loaded this skill. The harness seems to filter
cookies by some criterion (HttpOnly flags, parent-process boundary) that
drops the visible auth cookies.

This means **empty `getAllCookies` ≠ expired session**. The agent must run
this diagnostic BEFORE attempting any login, to avoid wasting clicks AND
to avoid risk-locking the account.

## Run order

```python
# 1. Navigate to feed
new_tab("https://app.seesaw.me/")
wait_for_load()

# 2. Check session via sessionStorage + DOM markers
result = cdp('Runtime.evaluate', expression='''
JSON.stringify({
    accountSwitcher: sessionStorage.getItem("accountSwitcher:activeAccount"),
    bodyHasUser: document.body.innerText.includes("Robert Hsiung"),
    bodyHasFamily: document.body.innerText.includes("Family"),
    url: location.href
})
''', returnByValue=True)
state = json.loads(result['result']['value'])

# 3. Decision
if state['bodyHasUser'] and state['bodyHasFamily'] and '/family/feed' in state['url']:
    print("[OK] Session is alive. No login needed.")
    # Proceed to feed extraction
else:
    print("[DEAD] Session expired. Loading saved cookies...")
    # Load seesaw-cookies.json, set via Network.setCookie
    # Or stop and ask Robert to log in
```

## Fallback: `document.cookie`

If the sessionStorage probe is inconclusive, `cdp('Runtime.evaluate',
expression='document.cookie', returnByValue=True)` returns the
document-visible cookies. A live Seesaw session shows:

- `g_state={"i_l":0,"i_ll":...,"i_b":"..."}` — Google sign-in state
- `G_AUTH2_MIGRATION="informational"`
- `_xsrf="2|..."` — Tornado XSRF token
- `stateParam="..."`
- `AMP_*=...` — Amplitude analytics

If `g_state` and `_xsrf` are both present for `app.seesaw.me`, the
session is valid.

## Pair with Pitfall #12

This diagnostic + Pitfall #12 (Family Member Sign In at y=341 viewport,
NOT y=380 vision) together explain every "the login failed again"
symptom in the skill. The previous "4 failed login attempts in one
session" narrative was caused by clicking at vision-reported y=380,
which missed the button entirely. **First verify the session, then
click at viewport coords — never at vision coords inside forms.**

## Cron-context note

For the 8PM cron path (`seesaw-photos-wechat`, job `40ef4787548e`),
this diagnostic is the FIRST step before invoking
`seesaw_photos_pipeline.py --check-cookies`. If the live DOM probe
already shows a healthy session, you can skip the cookie-expiry
check entirely — no need to load/save cookies on every run.

## Companion files

- `seesaw-cookies.json` (~/.hermes/profiles/influencer/) — saved cookies
  after a manual login; used only when the session IS actually expired
- `references/cron-photos-pipeline.md` — full 8PM cron recipe
- Skill Pitfall #9 (reset_password redirect) and #12 (button coords)