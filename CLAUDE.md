# 蔚瑄的英語甜甜圈屋 (Wei-Xuan's English Donut House)

An HTML web app for Vicky (蔚瑄) to practice English vocabulary and grammar, built for her
mom Abby. Gamified with stars, combos, daily streaks and real-world rewards (allowance,
stationery shopping) that Abby pays out manually. The app logic lives in one HTML file;
the content (vocab/grammar/reference) lives in three companion JSON files fetched at
runtime, so editing app logic doesn't require re-outputting the whole question bank.

## Where things live

- **Source file**: `english-forest.html` in this folder. Written as an **Artifact-style
  fragment** — no `<!DOCTYPE>`/`<html>`/`<head>`/`<body>` wrapper. Starts directly with
  `<title>`, a Google Fonts `<link>`, then `<style>`.
- **Content data**: `vocab.json`, `grammar.json`, `reference.json`, also in this folder.
  Loaded via `fetch()` at startup (`loadContentData()` near the top of the inline
  `<script>`) — the app waits on all three before rendering anything. See "Content model"
  below for what's in each. These must sit **next to** `english-forest.html` wherever it's
  served (same relative path), on every one of the three targets below.
- **Claude Artifact** (private, Abby's account): https://claude.ai/code/artifact/fc65db85-cb7f-4ffb-a28a-19886f8ffe90
  — update by publishing `english-forest.html` **and** the three JSON files together via
  the Artifact tool's multi-file `files` parameter (it auto-wraps the HTML with
  doctype/head/viewport; the JSON files are published as companion files referenced by
  `fetch()`, not inlined).
- **GitHub repo**: https://github.com/Abby1204/weixuan-english (public).
- **Live site** (what Vicky actually uses, bookmarked on her tablet):
  https://abby1204.github.io/weixuan-english/
- **Cloud sync DB**: a **dedicated** Supabase project — `https://knjhbrrdquylhdodzrvq.supabase.co`
  — deliberately separate from Abby's other Supabase project that backs her `investment`
  dashboard (`../investment/`, project `tmjnlhvaquwuchyambip`). Single table
  `english_progress`, one row `id='vicky'`, storing `{save, wordStats, lessonOverrides,
  config, grammarStats}` as JSONB. **No auth, no RLS** — any device reads/writes freely by
  design (matches the anon-key-is-public-by-design model, same as Firebase client config).
  Note this is *runtime progress*, separate from the *content* JSON files above — content
  is static and shipped with the app, progress is per-device state synced live.

## Deploying a change

**This folder itself is the git checkout** of https://github.com/Abby1204/weixuan-english
(remote `origin`, branch `main`) — there is no separate deploy checkout anymore. An
earlier version of this doc said to use "a separate git-tracked checkout", but that path
was actually just an ephemeral per-session scratchpad directory that stopped existing once
that session ended, which is exactly the "I can't find this path" problem this section
used to cause. Fixed 2026-09-27 by `git init`-ing this folder directly against the repo.

GitHub Pages needs a real `<!DOCTYPE html>`/`<head>` (with viewport meta) wrapper that the
Artifact tool normally adds automatically but a plain static host doesn't. `index.html`
(the deployed file) is a **generated** wrapper around `english-forest.html` (the source
fragment) — regenerate it, don't hand-edit it:

```bash
cd "/d/Temp for Abby/Claude/weixuan-english"
SRC="english-forest.html"
OUT="index.html"
LINE=$(grep -n "^</style>$" "$SRC" | head -1 | cut -d: -f1)
{
  echo '<!DOCTYPE html>'; echo '<html lang="zh-Hant">'; echo '<head>'
  echo '<meta charset="UTF-8">'
  echo '<meta name="viewport" content="width=device-width, initial-scale=1">'
  sed -n "1,${LINE}p" "$SRC"
  echo '</head>'; echo '<body>'
  sed -n "$((LINE+1)),\$p" "$SRC"
  echo '</body>'; echo '</html>'
} > "$OUT"
git add index.html vocab.json grammar.json reference.json english-forest.html CLAUDE.md
git commit -m "..."
git push
```

`english-forest.html`, `CLAUDE.md`, and `.claude/launch.json` (a static-server config for
local testing — see below) are committed too, so the GitHub repo is now the full durable
copy of everything, not just the deployed artifact.

Then also republish the Claude Artifact from the fragment source file **plus** the three
JSON files as companion files (`files` param on the Artifact tool — no HTML wrapping
needed there, the tool does that). If only content changed (not app logic), only the
JSON file(s) need updating on each target — the HTML doesn't need to move.

## ⚠️ Testing caution — this file talks to production

`english-forest.html` has the real Supabase URL/key hardcoded. Opening it **anywhere**
(even a local preview) connects to the **live production database** — the same row
Vicky's real progress lives in. Before doing any interactive testing that answers
questions / earns stars / changes config, neutralize the cloud write path first:

Also: the content JSON files are loaded via `fetch()`, which browsers block from a bare
`file://` page (CORS). Testing locally needs a plain static HTTP server run from this
folder (e.g. `python -m http.server` or an equivalent Node one-liner), not double-clicking
the HTML file. `vocab.json`/`grammar.json`/`reference.json` must be reachable at their
relative paths from wherever `english-forest.html` is served.

A `.claude/launch.json` config (`weixuan-english-static`, port 8743) is committed here for
exactly this. In one long-running session (2026-09-27) the Browser pane's `preview_start`
kept ignoring it and reusing a stale cached dev-server association from earlier in that
same conversation (another project's Vite server, on port 5173, serving this folder as a
subpath) — if that happens, the actual served files are usually still reachable at
`http://localhost:<that-port>/weixuan-english/english-forest.html`; check `get_page_text`
first if the page looks blank, don't assume the tool is fully broken before checking that.

```js
window.pushCloudData = async function(){};
window.schedulePush = function(){};
```

`pullCloudData()` (read-only) is safe to leave running. If a mistake reaches production,
fetch the current row, diff it against what you expect, and correct precisely rather than
resetting to defaults (Abby's real stars/combo/streak must never be wiped).

## App structure

Five home-screen activities, all reading from a shared `save`/`config` state that syncs
to Supabase:

1. **單字卡 Flashcards** — random-order browsing (no fixed sequence, ⬅️/➡️ with history
   for back/forward), auto-plays pronunciation twice per card (normal speed, then 1.5s
   later at `FLASH_SLOW_RATE`), no scoring. Also has a manual **🔁 自動播放** toggle: reads
   each card as English word → letters spelled out one at a time (each its own utterance
   with a real ~150ms pause between letters, not punctuation-based pausing, which isn't
   reliable across TTS engines) → English word again → Chinese meaning (zh-TW voice), then
   auto-advances through a freshly-shuffled pass of the current lesson pool. Capped at
   `FLASH_AUTO_MAX_ROUNDS` (3) full passes as a fail-safe so it can't run unattended
   forever, then auto-stops; also stops on manual card navigation or leaving the screen.
2. **單字測驗 Word Quiz** — per-round question count selector (10/20/.../全部作答);
   wrong-answer weighting (`wordStats`) makes previously-missed words resurface more in
   random mode; "重考錯題" button retests just the missed ones.
3. **拼字遊戲 Spelling** — three difficulties (basic: scrambled-letter tiles + Chinese
   hint; inter: A–Z keyboard + blank-count hint, no Chinese; exam: A–Z keyboard, no hints
   at all) plus the same count selector.
4. **文法練習 Grammar** — question bank tagged by **both** `range` (`midterm`/`final`,
   matching her actual exam scope — she deliberately does *not* want per-lesson
   granularity here, only reviews grammar before big exams) and `type` (8 categories as of
   this writing: `past_tense`, `there_was_were`, `comparative`, `modal`,
   `preposition`, `used_to`, `be_past`, `adjective` — `did_question` existed early on but
   Abby deleted it and its questions). Both filters are chosen in a
   pre-round settings overlay; `type` is **multi-select**. `past_tense` deliberately
   merges regular and irregular verbs into one category — kept separate they'd let her
   guess the answer shape from the category alone, defeating the point.
5. **📋 補充資料 Reference** — **browse-only, never scored**. Two decks so far:
   `課程單字總表` (auto-generated live from `DEFAULT_LESSONS`/`vocab.json`, not authored
   content — it isn't a `reference.json` entry, so it automatically tracks whatever
   `vocab.json` currently contains, including after a course-level swap), and the
   `reference.json` deck entries (currently one: `Irregular Verbs 過去式小幫手`, a grouped
   chart from a real teacher handout — each word has its own speaker button, not one
   shared button, and each group gets a distinct token color). New teacher handouts get
   appended to `reference.json`; Abby explicitly does **not** want this content folded
   into the quiz/spelling reward pool.

## Content model

Vocab, grammar and reference content live in three JSON files fetched at startup
(`loadContentData()` in `english-forest.html`), **not** inlined in the HTML — this keeps
editing the app's logic cheap even as the question banks grow. All three are tied to the
current course level (`courseLevel` field, currently `"N7B"`): when Vicky moves up to
`N8B`, these get **replaced wholesale**, not appended to — no history of past course
levels needs to be kept, so there's no versioning/migration logic, just an overwrite of
the file(s). (`GRAMMAR_RANGES` — `midterm`/`final` — stays hardcoded in the HTML; it's a
semester-structure concept, not course content, and isn't expected to change with level.)

- **`vocab.json`**: `{courseLevel, totalLessons, lessons: {1: [...], ..., 18: []}}`.
  Loaded into `DEFAULT_LESSONS`/`TOTAL_LESSONS`. Lessons 1–8 filled from a real "N7
  Homework Word / Midterm Test Words" sheet (80 words after a Lesson-5 correction —
  duplicates across lessons like `river`/`catch` are intentional, matching the real
  homework). Lessons 9–18 start empty; addable via the in-app "✏️ 新增/編輯課程單字"
  editor (paste `english,中文` lines) or by Abby handing me a photo to structure — those
  edits go to `lessonOverrides` (still in Supabase, not this file) rather than mutating
  `vocab.json` directly.
- **`grammar.json`**: `{courseLevel, types: [{key,label,icon}, ...], questions: [...]}`.
  Loaded into `GRAMMAR_TYPES`/`GRAMMAR`. 300 questions as of this writing, all
  `range:"midterm"` so far, schema `{range, type, s, o[4], a, e}`. `types` moved into this
  file (rather than staying hardcoded like `GRAMMAR_RANGES`) specifically because Abby
  confirmed the category set itself changes per course level, unlike `range`. Abby curates
  both `types` and `questions` herself (pastes/edits entries directly in this file) rather
  than having another AI generate batches from a spec, to avoid burning tokens on it — the
  key rules still apply though: `range` ∈ {midterm,final}; `type` must be one of the keys
  declared in this file's own `types` list, never invented ad hoc, and a new type is only
  added if it *doesn't* leak the answer the way separate regular/irregular past-tense
  categories would have; `s` must be globally unique — it's the key used for the star-cap
  tracking.
- **`reference.json`**: `{decks: [{id, icon, title, colorTokens, groups, tip}, ...]}`.
  Loaded into `REFERENCE_DECKS`. Extensible array for future teacher handouts; currently
  just the Irregular Verbs deck. Also tied to course level per Abby (N8B will likely swap
  this too), but small enough that it isn't a token-cost concern the way vocab/grammar
  were — split out mainly for consistency with the "content swaps wholesale on JSON
  files" model.

## Reward system (all admin-configurable — see below)

- **Stars**: awarded **immediately per correct answer** (not batched at round end — an
  earlier design batched it, which meant quitting mid-round lost all progress; fixed).
- **Grammar-only star cap**: same grammar question can only earn stars the first
  `config.grammarStarCap` times (default 3) it's answered correctly — prevents
  memorizing answer positions from being profitable. Deliberately **not** applied to
  vocab (repetition there is legitimate spaced-repetition practice, not gaming).
  Tracked in `grammarStats[sentence].correct`.
- **Combo**: real consecutive-correct streak (`save.combo`, resets to 0 on any wrong
  answer across quiz/spelling/grammar), `save.bestCombo` is the all-time record.
- **Daily check-in**: `config.dailyGoalStars` (default 5) stars in a calendar day
  completes that day; `save.dailyStreak` counts consecutive completed days. A day gap is
  detected and the streak **zeroed immediately** on next load (`checkDailyStatus()`) —
  it used to only self-correct lazily on the next successful day, which showed a stale
  number in the meantime. Returning after ≥1 day away also swaps the home greeting to a
  "long time no see, N days ago" message once.
- **Tiers / badges**: `config.tiers` (default 0/60/180/400/800 stars → 甜甜圈新手 /
  甜點學徒 / 甜點師傅 / 甜點主廚 / 英語甜點大師). 800 is deliberately hard — meant to
  take most of a semester of real practice, not one cram session.
- **Real-world payouts** (Abby manually fulfills these — the app only tracks and
  celebrates): every `config.streakBonusDays` (default 5) days of daily-check-in streak
  → `config.streakBonusText` (default "30 元零用錢"); reaching the final tier →
  `config.finalRewardText` (default "200 元文具店自由選購", Abby has since changed this
  to 250 in real use). A celebratory popup fires automatically at each milestone.
- **🎁 獎勵說明**: a home-screen button showing the current rules + tier ladder with
  checkmarks for what's achieved, always reflecting live `config`.

## Modes & admin panel

- **蔚瑄模式 / 家長測試模式** toggle (top-right pill, no password) — Parent mode lets
  Abby freely test the app without any of it touching Vicky's real stars/combo/streak/
  grammarStats. A first-run modal forces picking one before the app is usable.
- **⚙️ 後台管理 (MASTER)** — only reachable from Parent mode, gated by
  `config.parentPassword` (Abby has changed it from the `0000` default — don't assume
  it). Lets her edit: daily goal stars, streak bonus days/text, final reward text, the
  full tier ladder (free-text `min,label` lines), grammar star cap, the password itself,
  and an optional password hint shown at the login prompt. Also has a **manual star
  adjustment** tool (+/- amount with an optional reason, clamped at 0, keeps the last 20
  adjustments in `save.starAdjustments` for Abby's own audit trail) — for correcting
  misbehavior or mistakes without touching the database directly.

## Known future plan

See the assistant's own persistent memory for full detail, but in short: **when Vicky
reaches the final tier, the plan is to re-theme the whole app** (new seasonal visual
identity — Abby's example was Christmas — new mascot, new tier ladder) rather than just
leaving her sitting at max level. Confirm the desired direction with Abby before building
it; don't do it unprompted.

## Visual identity notes

- Pink "甜甜圈屋" (donut shop) theme — palette tokens defined once in `:root` (light) and
  mirrored for `prefers-color-scheme: dark` / `data-theme="dark"`. An inline script forces
  `data-theme="light"` on load, so the app always renders the light palette regardless of
  the device's system theme — Abby's device auto-switching to the darker palette read as
  an unwanted color shift ("淡粉紅" turning "深粉紅"), and there's no in-app toggle to flip
  it back, so this pins it to light. The dark tokens are left in the CSS in case a manual
  light/dark toggle is wanted later. Semantic colors
  (`--ok` green for correct, `--coral` red for wrong) are intentionally separate from the
  `--pink` brand accent, plus `--sky`/`--sun`/`--lavender` as secondary "candy" accents
  used for the 5 home-screen activity icons.
- Mascot is a baby chick emoji (🐥), named **蛋黃酥** (a real Taiwanese pastry, punning on
  chick/egg) — went through owl → white cat (emoji can't actually be recolored, this was
  attempted via CSS filter) → turtle → chick before landing here; chick was chosen partly
  because its glyph is visually round/symmetric enough to actually look centered in the
  circular avatar frame, which the turtle wasn't.
- Fonts: "Baloo 2" (display/headings, playful/rounded) + "Noto Sans TC" (Chinese body
  text), loaded from Google Fonts.
- Supabase JS client loaded via `cdn.jsdelivr.net` (the only script CDN the Claude
  Artifact CSP allows besides cdnjs — don't switch to Supabase's own `gstatic`-style CDN,
  it'll be blocked there even though it'd work fine on plain GitHub Pages).
