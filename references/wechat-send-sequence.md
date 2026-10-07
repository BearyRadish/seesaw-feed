# WeChat photo-send for Seesaw posts — verified stages + the logout trap

Session: 2026-09-06 evening (Sep3 Powers of Ten 7-photo send). The full send
sequence was verified stage-by-stage UP TO the final Enter; the send itself was
blocked only by an unrelated WeChat logout (documented below as a trap to check
BEFORE pressing Enter). The WeChat-GUI base reference is the (user-owned)
`wechat-macos-automation` skill — Pitfalls #36-#55. This file records what this
session added on top.

## Stage 1 — Picker navigation (the part that kept failing before)

The failure across earlier attempts: `osascript keystroke "/long/path"` drops
characters AND splices into the Cmd+Shift+G field's existing autocomplete
content (target path got merged into `.../2026-08-31-happy-birthday-william/`).
Cmd+A inside the Go-to field triggers Apple+F (Find), not select-all.

**Working sequence (verified first-try landing):**

```bash
cliclick c:300,344          # 1. focus picker chrome (NOT the location bar)
sleep 0.5
osascript -e 'tell application "System Events" to keystroke "g" using {command down, shift down}'
sleep 1.5
printf '/Users/openclaw/workspace/sas-5cs/seesaw-photos/<folder>' | pbcopy   # 2. clipboard, never keystroke
sleep 0.3
osascript -e 'tell application "System Events" to keystroke "v" using {command down}'
sleep 1
osascript -e 'tell application "System Events" to keystroke return'
sleep 2.5
```

**Verification without vision** (vision_analyze was down this session):
colorful-pixel density in the picker's file grid — a 7-photo folder shows
~400K colorful pixels (max-channel − min-channel > 30 over the grid region);
the stale 2-file folder showed far less. AX also works: the sheet's static
texts / sidebar rows are readable, but the location-bar path itself is not
AX-exposed — pixels are the practical check.

## Stage 2 — Selection + Open (AX-click beats coordinate click)

Cmd+A on the file grid is unreliable (Pitfall #39). The robust path:

1. Click once inside the file grid (x≈1000+, i.e., right of the sidebar) to focus it.
2. Check `value of attribute "AXEnabled"` of the sheet's **Open button** —
   enabled=true means a selection exists (the icon grid exposes NO
   AXSelectedRows; the only AXOutline in the sheet is the 26-row sidebar).
3. **AX-click the Open button** — immune to the coordinate-click deselect bug
   (Pitfall #45 applies to cliclick/screen-coordinate clicks only):

```bash
osascript -e '
tell application "System Events"
    tell process "WeChat"
        repeat with e in (entire contents of sheet 1 of window 1)
            try
                if role of e is "AXButton" and title of e is "Open" then
                    click e
                    exit repeat
                end if
            end try
        end repeat
    end tell
end tell'
```

Verified: `count sheets of window 1` went 1 → 0 and the photos loaded into the
input (input-area colorful pixels jumped to ~63K).

## Stage 3 — Merge bubble (position SHIFTS after photos are staged)

With an empty input the toolbar icons sat at x=339/375/411/447/497 (y≈960-975);
after 7 photos staged, the toolbar re-flowed to 304/339/375/526/688 and the
merge bubble moved from the documented (363,960) to **(406,960)** — the ring
outline spans x=397-416 and the "Show as merged" text cluster x=427-529.

**Re-derivation recipe (no vision needed):**
- Scan y≈935-985, x=300-560 for mid-gray ring pixels (gray 90-170, ≥3 px/column).
- The ~20px-wide ring cluster immediately LEFT of the white text cluster is the bubble.
- Click its center with the move-away-then-back sequence (Pitfall #36).
- Verify ON: green pixels (g > r+15 & g > b+15) inside the ring — ≈205 = ON,
  ≈0-10 = OFF.

**Color truth for the Send button**: WeChat green is (7,193,96). A muted
(146,173,153) sampled "green" is illustration artwork — don't read it as
"Send is green".

## Stage 4 — Send + THE LOGOUT TRAP (check BEFORE pressing Enter)

Symptom chain observed: Enter did nothing; typing a test char produced ZERO
pixel change; `focused of window 1` = false; Escape did nothing; the input
area showed a solid ~190K-colorful-pixel wall from y≈600 down.

**Root cause: WeChat had logged out mid-flow.** The window was the login
screen — AX static texts read "Open WeChat" / "Switch Account" / (later)
"Scan to log in". Its colorful illustration fills the window and mimics a
photo-filled input area.

**Diagnostic sequence when keystrokes stop landing in WeChat:**
1. `osascript ... focused of window 1` → false is the early signal.
2. Dump AX static texts of window 1 — login-screen strings confirm logout.
3. STOP retrying Enter/Escape — they go nowhere, and staged input is LOST.

**Recovery**: phone-side QR scan. The QR renders near screen center
(~x=850-1130, y=370-650), OUTSIDE the window's AX bounds — coordinate clicks
derived from window geometry miss it; screencapture the region and hand it to
Robert. Note the login screen ALSO renders outside the AX window bounds
("Open WeChat" button reported at x=875 while the window spans x=90-790).

## Excluding metadata.json from the send

Don't fight picker multi-select deselection. Before opening the picker:

```bash
mv <photo-folder>/metadata.json /tmp/_<slug>_metadata.json
# ... send ...
mv /tmp/_<slug>_metadata.json <photo-folder>/metadata.json   # restore after
```

Then any select-all in the picker grabs only the photos.
