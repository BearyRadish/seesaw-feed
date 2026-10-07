# Caption + delivery policy — merged-only WeChat sends (Robert's rules, 2026-09-06)

Supersedes the per-image-caption send strategy documented in earlier sessions.
Robert's words: "I think it is still better to send all images as a single
merged post on WeChat. We don't want to spam the group with photos."

## Rule 1 — Multi-page posts are ALWAYS one merged WeChat message

Never send per-photo messages, even when every photo has its own real caption
(e.g., Sep3 Powers of Ten: 7 captions "Max and Albert playing…", "Aspen and
Jennie playing…", …). Those captions are INPUT to a summary, not per-bubble text.

## Rule 2 — Merged caption = shared activity + all tagged student names

Implemented in `seesaw_api.build_smart_captions_for_item()` (returns a
single-element caption list + `has_real_captions` flag):

1. Enrich the post via `/api/item_v2` to get real per-page captions.
2. Strip the leading name list from each caption to recover the shared activity
   text. **Validate candidate names against `people_tags` first names** — never
   guess from capitalization. Without validation the stripper eats "Happy"
   (from "Happy Birthday, William!") and "Sidewalk" (from "Sidewalk &
   Sprinklers 2026!") as names. `_strip_student_names(caption, known_names)`
   implements the validated form.
3. Compose: `<activity> Featuring <names>.`
   - ≤6 students: list all ("Albert, Angela, Aspen, Darren, Evelyn, and Irene")
   - >6 students: first 5 + "and N others." ("Albert, Angela, Aspen, Darren,
     Evelyn, and 12 others.")
4. No real captions on any page → use the teacher comment
   (`extract_teacher_comments` filters "Thanks!" reactions, teacher-first,
   longest-first). No comment either → post `text` → generic
   "Class photo — see Seesaw for details".
5. Companion text message after the photo bubble:
   `<caption>\n— <author> (<date>)\n📎 <permalink>\n\n💬 Parents: please open
   Seesaw and leave an encouraging comment for your child!`

Verified outputs (2026-09-06):
- Sep3 (7 pages, real captions): `playing a "Powers of Ten Chutes & Ladders"
  game to practice using exponents during our math class today. Featuring
  Albert, Angela, Aspen, Darren, Evelyn, and 12 others.`
- Sep4 (4 pages, no captions, teacher comment): full Signify exhibition comment.

## Rule 3 — Aspen: exclude as AUTHOR, include as TAGGED student

Refinement of the family privacy rule. Only AUTHOR-based exclusion
(`people_tags_summary` == a student name / `is_teacher_post()` == False).
Aspen's first name appears in the "Featuring …" list like any classmate when
she is tagged in a teacher's class-activity photo. See SKILL.md privacy
section + `seesaw_api._all_tagged_students()` (keeps Aspen) vs the older
`extract_tagged_students()` (filtered her out — only for AUTHOR checks).

## Rule 4 — metadata.json never goes to WeChat

Robert: "we should try to avoid sending metadata.json if possible." Before
opening the picker:

```bash
mv <photo-folder>/metadata.json /tmp/_<slug>_metadata.json
# ... send ...
mv /tmp/_<slug>_metadata.json <photo-folder>/metadata.json   # restore after
```

Then Cmd+A / AX-Open selects only the photos. (This overrides
wechat-macos-automation Pitfall #55's "let it ride" guidance — that skill is
user-owned and can't be patched by the curator; this file is authoritative
for the Seesaw→WeChat pipeline.)

## Where this runs

- Interactive/cron helper: `seesaw_api.py` (`build_smart_captions_for_item`,
  `_summarize_captions_with_students`, `_strip_student_names`,
  `_all_tagged_students`, `extract_teacher_comments`) and
  `seesaw_photos_pipeline.py:compose_wechat_message` (merged-caption body).
- Photo staging: folder per post, `photo-NN-<uuid>.jpg` (no `.jpg.jpg` — strip
  the extension from the URL UUID segment before re-appending).
- The Quick-start pipeline snippet in SKILL.md still branches on
  `per_image` vs `merged` — that branch is now vestigial; ALWAYS use the
  single merged caption from `build_smart_captions_for_item()`.
- WeChat GUI mechanics for the send itself: see
  `references/wechat-send-sequence.md` (picker navigation, AX-Open, merge
  bubble at (406,960) when photos staged, logout trap).
