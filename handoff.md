# Handoff — เกมตกโน้ต (Spec 03 · Note Catcher)

> Written 2026-11-09 by the agent session that built it, so the next agent or human can continue without re-deriving context.
>
> - **Repo:** https://github.com/yos2568/note-catcher — public, branch `main`, commit `8ea889c`
> - **App:** `index.html` — one file, 62,410 bytes, vanilla JS, no dependencies, no network requests
> - **Spec:** `../../deploy-projects/specs/03-note-catcher-spec.md` (read it first; it is the contract)
> - **Series:** `01 Rhythm tap` (no repo yet) · `02 Practice Log ` → [practice-log](https://github.com/yos2568/practice-log) · **03 Note Catcher** (this one)

---

## 1. Snapshot

A treble/bass/alto/tenor note-reading arcade game for first-year Western-music performance majors (3500230, Faculty of Fine and Applied Arts, Chulalongkorn). A card with a mini staff falls toward a brass catch line; the player names the note before it lands. Three lives, rising speed, per-level best scores.

The spec was delivered in full, then extended twice at the instructor's request:

| Asked for | Added | State |
|---|---|---|
| Spec 3 of 3 (single-file game) | Everything in the spec | done |
| "add treble, bass, tenor, alto clef" | 4 clefs + ผสม (random per card) | done |
| "add accidentals incl. double sharp/flat" | ♮ ♯ ♭ 𝄪 𝄫 as a second input row | done |
| "add piano keyboard to input notes" | เปียโน input mode | done |
| "no keyboard display" | input switch moved under the field, works mid-game | done |
| "make the text A B C D easy to read" | note names on keys, theme-correct ink | done |
| "black key must include enharmonic" | C♯/D♭ stacked labels | done |

Nothing is uncommitted. `git status` is clean.

## 2. What the build does today

**Game loop.** One card at a time. `progress += dt / T` per frame (rAF, `dt` clamped to 50 ms), position = `progress × travel` so a resize never teleports the card. Next card spawns 0.6 s after a resolution. States: `idle → playing ⇄ paused → over`.

**Clefs** (bottom line → anchor line). `CLEFS` in the source:

| key | bottom line | glyph anchor | label |
|---|---|---|---|
| `treble` | E4 | G4, step 2 | เทรเบิล |
| `bass` | G2 | F3, step 6 | เบส |
| `alto` | F3 | C4, step 4 | อัลโต |
| `tenor` | D3 | C4, step 6 | เทเนอร์ |

**Levels are ranges of staff steps**, not pitches (`step 0` = bottom line). This is what makes one level work under all four clefs:

| level | steps | notes | T start → floor |
|---|---|---|---|
| ง่าย | −2…5 | 8 | 7.0 s → 2.6 s |
| กลาง | −2…9 | 12 | 6.0 s → 2.0 s |
| ยาก | −4…12 | 17 | 5.0 s → 1.6 s |

Treble ง่าย is therefore still C4–C5 exactly as the spec table says; bass ง่าย is E2–E3, alto D3–D4, tenor B2–B3.

**Accidentals.** `{natural 0, sharp +1, flat −1, dsharp +2, dflat −2}` applied to the clef pitch. Setting **เครื่องหมาย**: `ไม่มี` (pad hidden) · `ชาร์ป–แฟล็ต` (default) · `ทั้งหมด`.

**Two input modes** (**ตอบด้วย**):

- `ชื่อโน้ต` — arm an accidental (♮ ♯ ♭ 𝄪 𝄫, keys `1`–`5`), then press the letter (keys `a s d f g h j`). **Spelling matters**: C♯ ≠ D♭.
- `เปียโน` — one octave, 7 white + 5 black keys (keys `a w s e d f t g y h u j`, DAW layout). **Pitch matters, spelling does not**: a D♭ card is correctly answered on the C♯/D♭ key, and the feedback still names the written note. This is the honest behaviour — a piano has one key there.

The input switch is deliberately **not** disabled during play (`bindSeg(seg, key, alwaysOn)` third argument).

**Panels.** Idle / paused / over, all in-page. Over shows score, best per level, `สถิติใหม่!`, correct count, accuracy, and up to 3 most-missed pitch class + accidental combinations.

**Audio.** Web Audio only, created on the first gesture. Master gain `0.25 × volume` (default 60%). Correct note = sine, 10 ms attack / 340 ms hold / 150 ms release at `440 × 2^((midi−69)/12)` of the *written* octave. Wrong = triangle 110 Hz, 160 ms. Game over = G4 then C4. New best = C5 E5 G5 arpeggio after the panel. `ทดสอบเสียง` = A4.

**Persistence.** `note-catcher-settings-v1` = `{level, mode, names, clef, acc, input, volume}`; `note-catcher-best-v1` = `{easy, medium, hard}`. Both reads and writes are wrapped in `try/catch` with an in-memory fallback; when storage is blocked the notice `เบราว์เซอร์นี้เก็บคะแนนสูงสุดไม่ได้ เกมยังเล่นได้ตามปกติ` appears once and the game keeps working.

## 3. Rules that are easy to get wrong

- Multiplier is computed **before** the combo increment: `mult = min(4, 1 + floor(combo/5))`, then `combo++`. ×2 starts on the 6th consecutive correct answer.
- A wrong letter costs **one life per card**; the card keeps falling and can still be answered correctly for points, but the life stays lost. Second wrong press on the same card is ignored (`card.penalized`).
- A miss costs a life **unless** the card is already penalized — then only the combo resets.
- Speed drops 0.15 s per correct answer and never below `tmin`.
- Any life loss zeroes the combo. Reaching 0 lives ends the game immediately, even mid-fall.
- Hidden tab or window blur pauses; resuming is manual (`เล่นต่อ` / `p` / space).

## 4. Where to touch things (source map)

Single `<script>` IIFE, in this order: constants → DOM refs → storage → state → audio → note naming → clef glyphs → accidental glyphs → pitch math → staff SVG → pads → HUD → panels → options → spawn → rules → loop → start/pause/end → events → init.

Keys to know:

- `pitchFor(clef, step)` → `{midi, pc, octave}`. `chromaOf(pc, acc)` → 0–11, used only by piano mode.
- `staffSVG(clef, step, acc)` builds the card; viewBox `0 0 150 120`, staff `x 36…148`, notehead `cx 90` (60 %), accidental right-aligned at `x 78`, `cy = 84 − 6·step`.
- `clefGlyph()` / `accGlyph()` / `ACC_VB` return **drawn** SVG, never Unicode musical symbols.
- `syncSegments()` must list every segmented control — **add a new seg here or its selected state will silently freeze** (this bug already happened once with `accSeg`).
- `setOptionsDisabled()` locks the option rows during play; `inputSeg` is intentionally absent.
- The card carries `data-note`, `data-acc`, `data-clef`, `data-step` — the test harness reads these.

## 5. Deliberate deviations from the spec

Each was a judgment call; all are worth keeping unless the instructor says otherwise.

1. **Clefs and accidentals are drawn, not typed.** The spec bans 𝄞 because font coverage varies. The same logic applies to 𝄪/𝄫 (Musical Symbols block), so everything is SVG path work. Only ♯ and ♭ appear in *text*, and doubles are written `♯♯` / `♭♭` — both inside the universally-covered Miscellaneous Symbols block.
2. **Only the input switch is unlocked during play.** ระดับ / เคลฟ / เครื่องหมาย still lock because they change the game content; ตอบด้วย does not, because it is a control.
3. **Over panel is compacted**: `ตอบถูก … ข้อ` and `แม่นยำ …%` share a line and `สถิติใหม่!` is an inline badge. Without this the panel overflowed a 333 px field on a 360 × 640 phone (measured 350 px → 331 px).
4. **Answer pad bleeds 8 px into the gutter below 400 px** so seven 44 px targets fit one row at exactly 360 px. It wraps 4 + 3 below 360 px (the spec says below 340 px; 7 × 44 + 6 × 6 = 344 px simply does not fit 328 px of content box).
5. **Play field is `min(47svh, 420px)` below 360 px wide**, otherwise the wrapped pad pushed the second button row off a 320 × 640 screen. At 360 px and up it is the spec's `min(52svh, 420px)`.
6. **The paused panel uses a 97 % scrim** (idle/over stay at the spec's 92 %) so the frozen card does not ghost through the text.
7. **Piano keys are labelled** with note names and both black-key spellings. This makes the mode a *staff → key* drill rather than *staff → name → key*. Easy to undo: remove `.key-note` from `buildPiano()`.

## 6. How it was verified

No test files ship with the repo (the deliverable is one `index.html`). The harness was built ad hoc and lives in `/tmp` only — recreate it this way if needed:

1. Copy `index.html` and inject, **before** the game script, a `requestAnimationFrame` shim backed by `setTimeout(cb, 16)` plus a matching `cancelAnimationFrame`. Headless Chrome under `--virtual-time-budget` fires rAF once and then stalls; the shim is what makes virtual time drive the game.
2. Append an assertion script that drives the DOM only (`PointerEvent('pointerdown')` on `.pad button[data-pc]`, `#accPad button[data-acc]`, `#piano button[data-pc][data-acc]`), reads `#card` data attributes, and writes `RESULT:: PASS … || PASS …` into `document.title`.
3. Run: `chrome --headless=new --virtual-time-budget=600000 --dump-dom` and grep the title.

Current suite: **66 assertions, all passing** — geometry (all 17 steps, ledger lines, `cy = 84 − 6·step`), pitch mapping per clef (alto step 4 = C4, tenor step 6 = C4, bass step 0 = G2), all 5 accidentals appearing, wrong-accidental rejection, enharmonic acceptance in piano mode, keyboard maps for both input modes, multiplier maths, speed floor (measured 1600 ms), pause/resume without jumping, storage persistence and blocked-storage fallback, letter mode, 320/360/390 layout, and zero console errors.

Known flakes (test-side, not app-side): random sampling of 17 pitches occasionally misses one in ~120 cards, and assertions that assume `+10` break once the combo multiplier is ≥ ×2.

Manual checks that matter: open from `file://`, play one round per clef, switch input mid-card, rotate to landscape, toggle the OS theme.

## 7. Open issues / limits

- **GitHub Pages is not enabled.** `index.html` is at the repo root, so `gh api -X POST repos/yos2568/note-catcher/pages -f 'source[branch]=main' -f 'source[path]=/'` publishes it at https://yos2568.github.io/note-catcher/. Untested.
- **Letter mode ignores the clef visually** — the card shows only the note name, so the clef is invisible (the pitch/octave still follow it). Fine for the letter answer, useless for clef practice.
- **Piano keys are narrower than 44 px** on phones (37.7 px white / 23 px black at 320 px). Inherent to seven white keys on a 320 px screen; the keys are 112 px tall and black keys carry a 10 px-wide invisible hit extension.
- **Name mode needs two taps** for an accidental card. The alternative (12 chromatic buttons) was rejected because it would collapse the spelling test.
- **No key-press sound** on the piano. Cheap to add (`blip` already exists) but it was kept out to avoid competing with the correct-answer tone.
- **No README** in the repo, and `01 Rhythm tap` still has no repo of its own.

## 8. Commands

```bash
cd "/Volumes/Crucial2TB/All Codes/FAA/Arttechno/App for building/03 Note Catcher Spec"
open index.html                              # local check
git status && git add -A && git commit -m "…" && git push
gh repo view yos2568/note-catcher            # confirm remote state
```

Deploy targets the spec requires (GitHub Pages at the repo root, Netlify Drop, Vercel) all serve a bare `index.html` with no build step — drag the folder and you are done.

## 9. Suggested next steps

1. Enable Pages and open the live URL on a phone (the spec's acceptance criteria 1–15 were verified locally, not on a published URL).
2. Decide whether piano key labels should be a toggle (beginner aid vs. no-aid drill).
3. Consider one short round-trip on real student hardware: Windows lab PC with Chrome/Edge at 1366 × 768, and one iPhone.
4. If the series continues, mirror this file's structure into `01 Rhythm tap` and give it its own repo so all three apps look the same.
