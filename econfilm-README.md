# Economics & Film — build package

A digital-textbook + interactive-arena platform for **ECON 383D · Economics in Film** (Dr. Sharma, Spring 2026), built to mirror StrategyArena on the same Supabase backend.

---

## What's in this folder

### Front-end (open in any browser)
| File | What it is |
|---|---|
| `econfilm-index.html` | **The LMS shell.** Auth, dashboard, textbook launcher, arenas, practice, exams, assignments, gradebook, roster, analytics, certificate. A rebranded fork of StrategyArena's `index.html`, wired to the same Supabase project and pointed at the film curriculum. |
| `econfilm-curriculum.html` | **The 14-chapter digital textbook** (primer + 13 topics), with interactive figures. Opened per-chapter by the shell via `#chapter-N`. |
| `econfilm-<slug>.html` ×13 | **The interactive arenas** — one game per topic (see list below). Self-contained; each posts its score back to the shell. |
| `econfilm-schema.html` | The build blueprint you approved. |

### Server content (to load into Supabase)
| File | What it is |
|---|---|
| `econfilm-content.json` | **All authored content:** 78 chapter-quiz questions (with answer keys + explanations), the comprehensive final exam, and the 7 paper assignments with due dates. Source of truth for seeding the backend. |

### The 13 arenas
`commons` · `coase` · `deterrence` · `lemons` · `pow` · `gold-standard` · `bank-run` · `depression` · `csr` · `subprime` · `crime` · `capital` · `unions`

---

## How the pieces connect

1. A student signs in to `econfilm-index.html` and joins the course with a code.
2. **Textbook** cards open `econfilm-curriculum.html#chapter-N` in a new tab.
3. **Arenas** open `econfilm-<slug>.html`; on finish, each posts `{type:'econfilm-arena-result', slug, score}` back to the shell, which records the attempt (first two attempts count toward the arena component).
4. **Practice/quizzes** and the **final exam** are server-graded — questions and answer keys live in Supabase (from `econfilm-content.json`); correct answers never reach the browser.
5. **Papers** are submitted through the Assignments view and graded by the instructor (with the AI-assist rubric tool).

All files should sit in the **same directory** (or be served together) so the relative links resolve.

---

## The backend (namespace: `ef_`, distinct from StrategyArena's `sa_`)

The Economics & Film backend is a **complete, self-contained schema** namespaced `ef_*` so it lives alongside StrategyArena's `sa_*` in the same Supabase project with zero collision. The shell is already wired to it (every table, RPC, and edge-function reference uses `ef_` / `ef-`).

| File | What it does |
|---|---|
| `econfilm-backend.sql` | 26 `ef_` tables + gradebook view, RLS on every table (the answer-key vault `ef_assessment_item_keys` has **no** select policy — only the edge functions can read it), 3 helper functions, and all **27 RPC functions** matching the shell's calls. Validated against PostgreSQL 16. |
| `econfilm-seed.sql` | Seeds the course, 14 chapters, **82 questions + locked answer keys**, the comprehensive final (13 random-by-chapter + 4 fixed integrative items), and the 7 papers with due dates. Deterministic UUIDs — safe to re-run. |
| `supabase/functions/ef-grade-attempt/` | The grading engine (service role). Actions: `issue`, `grade`, `resume`, `regrade_numeric`, `propose`. Reads the answer vault server-side; correct answers never reach the browser. |
| `supabase/functions/ef-ai-grade/` | Instructor-assisted grading of written answers (`finalize`, `finalize_all`); uses the instructor's own AI key from `ef_grader_keys`. |
| `supabase/functions/ef-notify-prof-request/` | Records instructor-access requests (optional email via Resend). |

### Deploy order (needs your Supabase access)

1. **Run `econfilm-backend.sql`** in the Supabase SQL editor (schema + RLS + functions).
2. **Run `econfilm-seed.sql`** (course + all content).
3. **Deploy the edge functions:**
   ```
   supabase functions deploy ef-grade-attempt       --no-verify-jwt
   supabase functions deploy ef-ai-grade            --no-verify-jwt
   supabase functions deploy ef-notify-prof-request --no-verify-jwt
   ```
   They read `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ANON_KEY` (injected automatically). `ef-notify` optionally uses `RESEND_API_KEY` + `EF_NOTIFY_TO`.
4. **Claim ownership** so you can manage the course — after signing in once, run the one-line `update ef_courses set owner_id = …` shown as a comment near the top of `econfilm-seed.sql`.
5. **Buckets:** the shell reuses the shared Storage buckets `assignment-files` and `proctor-snapshots` (paths namespaced by course). Create them in Storage if absent.

Share instructor/admin login or project keys and I'll run all of this for you.

---

## Grading model — matches your syllabus exactly

The `ef_` schema resolves the weight-bucket gap that StrategyArena's `sa_` model had. `ef_courses` carries dedicated weights, seeded to your syllabus:

- **Papers (assignments)** → 60%, with `drop_lowest_assignments = 1` so the best 6 of 7 count
- **Final exam** → 30%
- **Attendance** → 5% · **Participation** → 5% (manual gradebook entries)
- **Reading / quizzes / arenas** → 0% (formative)

`ef_grade_breakdown()` computes the weighted total from these, and it was verified end-to-end on Postgres (e.g., a 9/10 paper yields the correct 54% assignments contribution). Attendance and participation are entered per student and surface via `ef_grade_overrides`.

---

## Visual identity

The entire platform — shell, textbook, and all 13 arenas — shares one dark palette with a **projector-gold** accent (the film identity), rebranded to **Economics & Film** with a gold film-clapper mark. The shell keeps StrategyArena's layout and component design for full structural parity, recolored from violet to gold throughout.

## Arena → gradebook integration

Each arena is a standalone file. On finish it posts `{type:'econfilm-arena-result', slug, score, played_at}` to the shell, which records the attempt to `sa_arena_results` (using the signed-in student's session and current course) — so arena scores flow into progress, badges, and the gradebook. Arenas opened directly (not via the shell) still play and show a score; they just don't record until launched from within the course.

---

*Built for Dr. Amarendra Sharma · Binghamton University · Department of Economics.*
