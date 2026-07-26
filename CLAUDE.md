# Sevenine IELTS Mock-Test Platform — Architecture & Working Notes

Single-file web app for Sevenine English School (Tashkent). One large HTML file +
Supabase backend. Deployed via GitHub Pages.

Brand: DM Serif Display + DM Sans, primary blue `#007AFF`, "Seve**n**ine" wordmark
(blue "n").

---

## THE MOST IMPORTANT RULE

Two independent surfaces, two different deploy models:

- **The HTML file** (`Sevenine_IELTS_Platform_27_12.html`) — changes go live ONLY
  after you commit + push to the GitHub Pages repo. Nothing you edit here affects
  students until you publish.
- **The Supabase database** — every write is live IMMEDIATELY.

Consequence: if you fix a grader/answer-key bug in the HTML but don't publish, the
live site keeps grading NEW submissions with the OLD logic. And any DB row you
"fixed" can be silently overwritten the next time the live (unpublished) code
re-grades it. When a fix touches both, do the DB patch AND ship+publish the file,
and say so explicitly.

---

## Backend (Supabase)

- Project ref: `typdiyoqusxopegguntv`. Access via Supabase MCP (`execute_sql`,
  `apply_migration`). AI grading runs through an Edge Function named `claude-proxy`.
- Tables: `tests`, `attempts`, `users`, `test_locks`, `calibration_bands`.
- PostgREST helpers in the file: `db(method,table,body,params)`, plus
  `dbGet/dbPost/dbPatch/dbDelete`. `SUPA_URL` / `SUPA_KEY` (anon) are inline.
- Large JSONB inserts: use dollar-quoting (`$json$...$json$::jsonb`), never manual
  quote-escaping. Use `jsonb_set` for surgical updates, `on conflict do update` for
  upserts.

### `tests` table — how a test is shaped
- Loaded by `loadSupabaseTests()` with `status=eq.published`, merged into
  `ALL_TESTS` (hardcoded `MOCK_TESTS` are the base; DB rows override by id).
- `test_type`: `full` | `reading_only` | `listening_only` | writing variants.
- A reading passage object MUST have: `id` (e.g. `"p1"`), `title`, `q_range`
  (e.g. `[1,13]`), `body` (HTML), `questions_html` (HTML). Plus top-level
  `answer_key`. **Omitting `id` or `q_range` makes Start render blank.**

### `attempts` table
Columns incl. `listening_answers/reading_answers/writing_answers` (jsonb),
`*_score` numerics, `listening_results/reading_results` (jsonb, per-question
`{correct,student,isCorrect}` + `_raw` + `_total`), `listening_raw/reading_raw`,
`status`, `submitted_at`, `graded_by/at`, `teacher_feedback`, `tab_switches`.
The results screen reads `*_results._raw` for the "X/40" — so when patching a
score, rewrite `_results` (incl. `_raw`), `*_raw`, AND `*_score` together so they
agree, and match the grader's native output format so a future re-grade won't
shift the display.

---

## Reading rendering — KNOWN TRAP

`getReadingQuestionsHTML(passageId, ...)` decides how a test's reading renders:
- A **generic DB branch** (excludes ids starting `mock-` / `cam14-`) renders from
  `reading_data.passages[].questions_html`. This is what new DB tests use.
- `mock-001` reading does NOT render from its `reading_data`. It renders from a
  **hardcoded block** inside `getReadingQuestionsHTML` ("Default: Mock Test 1").
  If you change mock-001 reading, edit that hardcoded HTML, not the reading_data.

`mock-001` == the test staff call **"Academic Purpose Test 2"** (its title is
"Mock Test 1"). The object literally id'd `academic-purpose-test-2` is a DIFFERENT
(Cambridge) test. Don't confuse them.

---

## Grading internals (client-side)

- `normaliseAnswer` / `acceptableAnswers` / `answerMatches`: text answers are
  lowercased/trimmed; `/` in a key = accepted alternatives; `()` = optional words;
  leading a/an/the optional. Units/symbols NOT stripped.
- Multi-select answer-key conventions DIFFER by section:
  - **Reading** (`autoScoreReading`): groups come **only from answer_key keys
    containing a hyphen** (e.g. `"14-16": "C, E, F"`). Single keys like
    `14`,`15`,`16` for an ms group read empty and score 0.
  - **Listening** (`autoScoreListening`): the OPPOSITE — keys must be
    **per-question singles** (`15:'A',16:'D'`); groups are inferred from the
    student's `ms_<lo-hi>` answers. A hyphen key in a listening answer_key is
    never matched and scores 0.
- MC `data-q` accepts `"l11"`, `"lq11"`, or `"11"` (parser strips `l`/`lq`).
  Anything else silently drops the click — pickMC saves nothing.
- `rawToBand(raw, table)` uses `READING_BANDS` (== `LISTENING_BANDS`), calibrated
  for a 40-question paper. **For 13-question mini readings the band value is
  meaningless — report raw X/13, ignore the band.**

### Multi-select ("choose TWO/THREE") checklist
Every `.ms-opt` needs `data-v`, `data-group="<lo-hi>"`, `data-max="<N>"`,
`onclick="pickMS(this)"`; wrap in `.ms-options data-group="<lo-hi>"`; counter id
`ms-count-<lo>`. AND the answer_key must use the group key `"<lo-hi>":"X, Y, Z"`.
Missing `data-group` → saved as group `"default"` → never matched → 0.
Missing `data-max` → selection cap breaks (students can over-pick).

### MCQ
`.mc-options[data-q="N"]` containing `.mc-opt[data-v="A".."D"]` with
`onclick="pickMC(this)"`. Answer_key `"N":"A"`.

---

## Test locks (`test_locks` table)

`{test_id, lock_hash}` where `lock_hash = sha256(password)`. Locks are DB-backed:
`loadTestLocks()` syncs from DB on dashboard load; `lockTest()` writes/deletes the
DB row (and mirrors to a `sn_test_locks` localStorage cache). To force-unlock a
test, delete its `test_locks` row. (Legacy note: locks used to be localStorage-only,
which is why old ones desynced.)

---

## Editing workflow (do this every time)

1. Work on a local copy of the HTML. Reading a specific area? Grep for the function
   name rather than trusting line numbers (the file is ~10k lines and shifts).
2. After any `<script>` edit, syntax-check every script block before shipping:
   ```bash
   node -e 'const fs=require("fs"),vm=require("vm");const h=fs.readFileSync(process.argv[1],"utf8");
   const re=/<script\b[^>]*>([\s\S]*?)<\/script>/gi;let m,i=0,bad=0;
   while((m=re.exec(h))){i++;const c=m[1];if(/^\s*$/.test(c))continue;
   if(/\b(import|export)\b/.test(c)&&!/function|var|let|const|=>/.test(c))continue;
   try{new vm.Script(c)}catch(e){bad++;console.log("ERR #"+i,e.message)}}
   console.log("blocks",i,"errors",bad)' Sevenine_IELTS_Platform_27_12.html
   ```
3. DB changes: verify with a read-back `select` after every write.
4. To deploy the HTML: commit + push to the GitHub Pages repo. State clearly in
   your summary which changes are already live (DB) and which need the publish.

## Preferences
Direct, concise, working deliverables over explanation. Honest pushback over
rubber-stamping. Calibrated, honest band assessment (rights to official IELTS
descriptors). Flag judgment calls (e.g. over-selection policy) rather than
deciding silently.
