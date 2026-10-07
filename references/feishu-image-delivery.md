# Delivering Seesaw Photos to Robert

After extracting photos via the parent browser skill, you need to get them
to Robert. Options ranked by reliability:

## 1. Drag from Finder (recommended, ~30 seconds)

Robert drags the photos himself from `~/workspace/sas-5cs/seesaw-photos/<slug>/`
to the WeChat input field. WeChat accepts bulk image drops — all 7 photos
upload in one batch. This is **the only reliable macOS path** for getting
multiple images into a WeChat chat.

**Privacy reminder:** these are photos of minors (Aspen's classmates).
Never auto-forward to other chats or platforms without explicit consent.

## 2. WeChat paste via clipboard (broken — known limitation)

See `wechat-macos-automation` Pitfall #27: even when clipboard contains a
JPEG, `osascript keystroke "v" using {command down}` does NOT paste the
image into WeChat's input field. Don't waste cycles retrying.

## 3. WeChat Send File icon (broken — known limitation)

The icon at (~190, 865) on Robert's Mac mini is the file/folder icon in the
input toolbar. Clicks register (notification badge changes) but no file
picker dialog opens. Likely the same Accessibility permission issue as #2.

## 4. WeChat "Photos and Videos" menu (wrong dialog)

`Window > Photos and Videos` opens the **chat's attachment history viewer**,
NOT a file picker. If you see a window titled like "棒呆留学..." — that's
the past attachment browser, not a send action. Close it (X button at
top-right).

## 5. iMessage fallback (clean CLI but different platform)

If Robert is OK with receiving via iMessage instead of WeChat, use the
`imessage` skill (CLI-based, supports image attachments reliably). The
trade-off is platform fragmentation — he gets the photos on his Mac's
Messages app, not his WeChat.

## 6. AirDrop (manual)

```bash
open ~/workspace/sas-5cs/seesaw-photos/<slug>/
# Opens Finder. Robert then AirDrops to his iPhone.
# From iPhone WeChat → forward to self.
```

Friction: 1-2 minutes + extra hops. Only worth it if drag-from-Finder
doesn't work.

## Decision tree

```
Can Robert receive the photos via WeChat himself (drag-from-Finder)?
  YES → just give him the file paths, done. ~30s.
  NO  → Use iMessage fallback. Different platform but reliable.
  NO  → Use AirDrop-via-iPhone. ~2 min extra.
```

Don't try to automate the WeChat image-send path. The known pitfalls
outnumber the working approaches, and the photos contain minors' faces —
the failure cost is real (broken messages, privacy leaks), while the
savings is ~25 seconds vs. drag-from-Finder.