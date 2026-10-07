# Seesaw Login Button Coordinates (verified 2026-09-04)

## Why this file exists

On 2026-09-04 the agent made 4 "failed Seesaw login attempts" because
the click coord for "Family Member Sign In" was wrong — vision_analyze
placed it at y≈380 in the screenshot, but `getBoundingClientRect()`
returned **y=341 (viewport coords)**. The y=380 click hit empty space
below the button. Each failed click did NOT register as a Sign In
attempt, so Seesaw didn't increment any "incorrect password" counter —
but the form got reset by an onBlur handler, which the agent then misread
as "the password was rejected." Robert had to correct the agent and
provide the password again, losing ~15 minutes.

## The lesson

**For login forms (or any click target inside a frame), ALWAYS query
`getBoundingClientRect()` via JS rather than trusting vision-reported
coords.** Vision_analyze on full screenshots can be 38px+ off in y for
elements at the bottom of forms.

## Verified coords (Robert's Mac mini, headless Chromium)

| Element | JS-discovered viewport coord | Vision-reported (wrong) |
|---|---|---|
| Email input `input[name=email]` | (892, 190.5) | matches |
| Password input `input[name=password]` | (892, 277.7) | matches |
| **Family Member Sign In button** | **(892, 341.7)** | y≈380 |
| Google SSO button | (830, 496.5) | matches |

## Working login sequence

```python
# 1. Navigate to login page (Family Member path)
new_tab("https://app.seesaw.me/#/login?role=parent")
import time; time.sleep(3)

# 2. Discover coords via JS (NEVER skip this for buttons)
result = cdp('Runtime.evaluate', expression='''
JSON.stringify((function() {
    const btn = Array.from(document.querySelectorAll('button'))
        .find(b => b.textContent.includes('Family Member Sign In'));
    const r = btn.getBoundingClientRect();
    return {x: r.left + r.width/2, y: r.top + r.height/2};
})())
''', returnByValue=True)
import json
coords = json.loads(result['result']['value'])
print(f"Sign In button at viewport ({coords['x']}, {coords['y']})")

# 3. Fill form (fill_input already handles selector-to-element resolution)
fill_input('input[name="email"]', os.environ['SEESAW_EMAIL'])
time.sleep(0.3)
fill_input('input[name="password"]', '<password here>')
time.sleep(0.5)

# 4. Click the JS-discovered coord (NOT vision coord)
click_at_xy(coords['x'], coords['y'])
time.sleep(5)

# 5. Verify success: URL should be /#/family/feed
url = cdp('Runtime.evaluate', expression='window.location.href', returnByValue=True)
print(f"URL: {url['result']['value']}")  # expect: https://app.seesaw.me/#/family/feed
```

## Failure detection

| Symptom | Diagnosis |
|---|---|
| URL = `/#/login?role=parent` (no change) | Click missed the button (re-check coords) |
| URL = `/#/reset_password` | Password was wrong (or rate-limited after multiple failed attempts — see SKILL.md Pitfall #9) |
| Form re-renders empty after click | Either click missed (above) OR Seesaw accepted but reset for unknown reason — re-query the form state via JS |