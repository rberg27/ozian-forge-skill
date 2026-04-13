# Forge Learning System — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build three Claude Code skills (`/forge`, `/forge-quiz`, `/forge-review`) that turn assignments into Bloom's-scaffolded Socratic learning sessions, create targeted flashcards in SQLite, and quiz the user daily via spaced repetition.

**Architecture:** Each skill is a `SKILL.md` file in `~/.claude/skills/<name>/`. Skills use Bash to run SQLite commands against `forge/flashcards/forge.db`. The forge data directory lives at `C:/Users/rberg/Documents/projects/ozian/forge/`. Skills read/write markdown files for assignments, scaffolds, gaps, and sitting logs.

**Tech Stack:** Claude Code skills (SKILL.md markdown), SQLite3 (CLI), Bash, Claude Code scheduled tasks.

---

## File Structure

```
~/.claude/skills/
├── forge/
│   └── SKILL.md                    # /forge skill — intake + Socratic dialogue
├── forge-quiz/
│   └── SKILL.md                    # /forge-quiz skill — daily spaced repetition quiz
└── forge-review/
    └── SKILL.md                    # /forge-review skill — browse progress

C:/Users/rberg/Documents/projects/ozian/forge/
├── assignments/                    # Created by /forge on first use
├── flashcards/
│   └── forge.db                    # Created by /forge on first use
└── docs/
```

---

### Task 1: Initialize Project Structure and SQLite Database

**Files:**
- Create: `C:/Users/rberg/Documents/projects/ozian/forge/flashcards/.gitkeep`
- Create: `C:/Users/rberg/Documents/projects/ozian/forge/assignments/.gitkeep`

This task sets up the base folder structure and verifies SQLite3 is available.

- [ ] **Step 1: Create the directory structure**

```bash
mkdir -p "C:/Users/rberg/Documents/projects/ozian/forge/assignments"
mkdir -p "C:/Users/rberg/Documents/projects/ozian/forge/flashcards"
touch "C:/Users/rberg/Documents/projects/ozian/forge/assignments/.gitkeep"
touch "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/.gitkeep"
```

- [ ] **Step 2: Verify SQLite3 is available**

Run: `sqlite3 --version`
Expected: A version string like `3.x.x ...`

- [ ] **Step 3: Create the SQLite database with the flashcards schema**

```bash
sqlite3 "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" <<'EOF'
CREATE TABLE IF NOT EXISTS flashcards (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    assignment    TEXT NOT NULL,
    concept       TEXT NOT NULL,
    bloom_level   TEXT NOT NULL,
    question      TEXT NOT NULL,
    answer        TEXT NOT NULL,
    teaching_note TEXT,
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
    next_review   DATETIME NOT NULL,
    interval_days INTEGER DEFAULT 1,
    ease_factor   REAL DEFAULT 2.5,
    repetitions   INTEGER DEFAULT 0,
    last_reviewed DATETIME
);
EOF
```

- [ ] **Step 4: Verify the database was created correctly**

Run: `sqlite3 "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" ".schema flashcards"`
Expected: The CREATE TABLE statement from above.

- [ ] **Step 5: Initialize git repo and commit**

```bash
cd "C:/Users/rberg/Documents/projects/ozian/forge"
git init
git add assignments/.gitkeep flashcards/.gitkeep flashcards/forge.db docs/
git commit -m "feat: initialize forge project structure with SQLite database"
```

---

### Task 2: Create the `/forge` Skill — Assignment Intake

**Files:**
- Create: `C:/Users/rberg/.claude/skills/forge/SKILL.md`

This is the main skill. This task covers the intake flow only (reading assignments, generating slugs, creating scaffold). The Socratic dialogue is Task 3.

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p "C:/Users/rberg/.claude/skills/forge"
```

- [ ] **Step 2: Write the `/forge` SKILL.md with intake logic**

Create `C:/Users/rberg/.claude/skills/forge/SKILL.md` with this content:

````markdown
---
name: forge
description: >
  Personal knowledge system. Takes assignments (text or PDF), scaffolds concepts
  using Bloom's taxonomy, runs Socratic dialogue to find knowledge gaps, teaches
  missing concepts, and creates flashcards for spaced repetition.
  Use when: "forge", "new assignment", "learn", "study", "forge continue", "forge status"
---

# Forge — Personal Knowledge System

You are a Socratic tutor that helps the user discover what they know and don't know.

**Base directory:** `C:/Users/rberg/Documents/projects/ozian/forge`
**Database:** `C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db`

## Routing

Parse the user's invocation to determine the mode:

- `/forge` or `/forge new` → **Intake Mode** (below)
- `/forge continue` → **Continue Mode** (resume Socratic dialogue — see "Socratic Dialogue" section)
- `/forge continue <slug>` → **Continue Mode** for a specific assignment
- `/forge status` → **Status Mode** (see "Status Mode" section)

If no subcommand, default to **Intake Mode**.

---

## Intake Mode

### Step 1: Get the assignment

Ask the user: "What's the assignment? You can paste text directly or give me a file path (PDF or text file)."

### Step 2: Process the assignment

- If a file path is given, read it using the Read tool (works for PDFs too)
- If text is pasted, use it directly

### Step 3: Generate slug and create folder structure

Generate a short kebab-case slug from the assignment topic (e.g. `rest-api-design`, `distributed-systems-hw3`).

```bash
SLUG="<generated-slug>"
BASE="C:/Users/rberg/Documents/projects/ozian/forge"
mkdir -p "$BASE/assignments/$SLUG/sittings"
```

### Step 4: Save the assignment

Write the assignment content to `assignments/<slug>/assignment.md` using the Write tool.

### Step 5: Analyze and create scaffold

Analyze the assignment and identify every concept/skill the user needs to know. Organize them by Bloom's taxonomy level. Write `assignments/<slug>/scaffold.md` in this format:

```markdown
# Scaffold: <Assignment Title>

## Remember
- [ ] `not assessed` — <Concept>: <What they need to recall>
- [ ] `not assessed` — <Concept>: <What they need to recall>

## Understand
- [ ] `not assessed` — <Concept>: <What they need to explain>

## Apply
- [ ] `not assessed` — <Concept>: <What they need to demonstrate>

## Analyze
- [ ] `not assessed` — <Concept>: <What relationships they need to identify>

## Evaluate
- [ ] `not assessed` — <Concept>: <What trade-offs they need to judge>

## Create
- [ ] `not assessed` — <Concept>: <What they need to synthesize>
```

Not every Bloom's level needs entries — only include what the assignment actually requires.

### Step 6: Create empty gaps file

Write `assignments/<slug>/gaps.md`:

```markdown
# Knowledge Gaps: <Assignment Title>

*Gaps identified during Socratic dialogue sessions.*
```

### Step 7: Confirm and begin dialogue

Show the user:
- The assignment slug
- A summary of the scaffold (how many concepts at each Bloom's level)
- Ask: "Ready to start the Socratic dialogue? I'll work through these concepts starting from the basics."

Then proceed to the **Socratic Dialogue** flow.

---

## Socratic Dialogue

This is the core learning loop. Used after intake and when resuming with `/forge continue`.

### Resuming

When resuming (`/forge continue`):
1. If no slug given, list assignments with unassessed concepts and ask which to continue
2. Read `assignments/<slug>/scaffold.md` to find the next `not assessed` concept
3. Pick up from there

### The Loop

For each unassessed concept, starting from **Remember** level and working up:

**1. Probe** — Ask the user to explain the concept. Open-ended questions only, never multiple choice.
- Remember: "Can you tell me what [concept] means?"
- Understand: "Can you explain in your own words why [concept] works that way?"
- Apply: "Can you walk me through how you'd use [concept] in [scenario]?"
- Analyze: "What's the relationship between [concept A] and [concept B]?"
- Evaluate: "What are the trade-offs between [approach A] and [approach B]?"
- Create: "How would you design a solution using [concept]?"

**2. Socratic dig** — Based on their response:
- If correct and complete → mark as `understood`, move on
- If partial or incorrect → ask 2-3 targeted follow-up questions to map exactly what they know vs. don't know. The goal is to find the precise boundary of their understanding.

**3. Teach** — Once the specific gap is identified:
- Explain the precise nugget they're missing. Not a lecture — a targeted explanation.
- Keep it concise and connected to what they DO already know.

**4. Verify** — Ask them to restate the concept in their own words to confirm the teaching landed.

**5. Record** — Based on the outcome:

If **understood** (got it right from the start):
- Update scaffold.md: change `not assessed` to `understood` for that concept

If **gap found and taught**:
- Update scaffold.md: change `not assessed` to `gap` for that concept
- Append to `gaps.md`:
  ```
  ## <Concept> (<Bloom's Level>)
  **Gap:** <What they didn't know>
  **Teaching:** <What was explained>
  **Verified:** <yes/no — did the verification check pass?>
  ```
- Create flashcard in SQLite:
  ```bash
  sqlite3 "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
    "INSERT INTO flashcards (assignment, concept, bloom_level, question, answer, teaching_note, next_review) VALUES ('<slug>', '<concept>', '<bloom_level>', '<question>', '<answer>', '<teaching_note>', datetime('now', '+1 day'));"
  ```

**6. Log** — After each concept (or when the user stops), append the dialogue to `sittings/YYYY-MM-DD.md`:

```markdown
---
## <Concept> — <HH:MM>

**Probe:** <the question asked>

**User response:** <summary of what they said>

**Follow-ups:** <summary of Socratic dig>

**Outcome:** understood | gap
**Gap detail:** <if gap — what was missing>
**Teaching:** <if gap — what was taught>
**Verified:** <yes/no>
```

**7. Repeat** — Move to the next unassessed concept. Ask the user if they want to continue or stop for now.

### Important Rules

- **One concept at a time.** Never assess multiple concepts in parallel.
- **Bottom-up through Bloom's.** Don't probe "Evaluate" if "Remember" items in the same domain are still unassessed.
- **The user can stop anytime.** Save progress and they can resume with `/forge continue`.
- **Flashcard questions should capture the specific gap**, not generic definitions. If the user confused PUT and PATCH, the card should ask about the difference, not "define PUT."

---

## Status Mode

When the user runs `/forge status`:

1. List all assignment folders in `assignments/`
2. For each, read `scaffold.md` and count:
   - Total concepts
   - Assessed (understood + gap)
   - Gaps found
   - Not yet assessed
3. Query flashcard counts from SQLite:
   ```bash
   sqlite3 "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
     "SELECT assignment, COUNT(*) as cards, SUM(CASE WHEN next_review <= datetime('now') THEN 1 ELSE 0 END) as due FROM flashcards GROUP BY assignment;"
   ```
4. Present a summary table:
   ```
   Assignment              | Progress    | Gaps | Cards | Due
   rest-api-design         | 8/12 (67%)  | 3    | 5     | 2
   distributed-systems-hw3 | 2/20 (10%)  | 1    | 2     | 0
   ```
````

- [ ] **Step 3: Verify the skill file was created**

Run: `ls -la "C:/Users/rberg/.claude/skills/forge/SKILL.md"`
Expected: File exists.

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/rberg/Documents/projects/ozian/forge"
git add -A
git commit -m "feat: create /forge skill with intake and Socratic dialogue"
```

---

### Task 3: Create the `/forge-quiz` Skill

**Files:**
- Create: `C:/Users/rberg/.claude/skills/forge-quiz/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p "C:/Users/rberg/.claude/skills/forge-quiz"
```

- [ ] **Step 2: Write the `/forge-quiz` SKILL.md**

Create `C:/Users/rberg/.claude/skills/forge-quiz/SKILL.md` with this content:

````markdown
---
name: forge-quiz
description: >
  Daily spaced repetition quiz. Pulls due flashcards from the Forge database
  and quizzes the user. Self-graded (Phase 1). Updates SM-2 intervals.
  Use when: "forge quiz", "quiz me", "flashcards", "daily quiz", "study"
---

# Forge Quiz — Spaced Repetition

**Database:** `C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db`

## Flow

### Step 1: Check for due cards

```bash
sqlite3 -header -column "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
  "SELECT id, assignment, concept, bloom_level, question FROM flashcards WHERE next_review <= datetime('now') ORDER BY next_review ASC;"
```

If no cards are due, tell the user: "No cards due right now! Next review is on [date]." To find the next date:

```bash
sqlite3 "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
  "SELECT MIN(next_review) FROM flashcards WHERE next_review > datetime('now');"
```

If there are no cards at all, tell them: "No flashcards yet. Run `/forge` with an assignment to get started."

### Step 2: Present cards one at a time

For each due card:

1. Show the **question** and the **Bloom's level** and **assignment** for context
2. Say: "Take a moment to think about your answer, then tell me when you're ready to see it."
3. When the user is ready, show the **answer** and the **teaching note** (if any)
4. Ask: "How'd you do? (got it / didn't)"

### Step 3: Update the card based on grade

**If "got it" (correct):**

Calculate the new interval and update:

```bash
sqlite3 "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" <<EOF
UPDATE flashcards SET
  repetitions = repetitions + 1,
  interval_days = CASE
    WHEN repetitions = 0 THEN 1
    WHEN repetitions = 1 THEN 6
    ELSE CAST(interval_days * ease_factor AS INTEGER)
  END,
  ease_factor = MIN(2.5, ease_factor + 0.1),
  next_review = datetime('now', '+' || CASE
    WHEN repetitions = 0 THEN '1'
    WHEN repetitions = 1 THEN '6'
    ELSE CAST(CAST(interval_days * ease_factor AS INTEGER) AS TEXT)
  END || ' days'),
  last_reviewed = datetime('now')
WHERE id = <card_id>;
EOF
```

**If "didn't" (incorrect):**

```bash
sqlite3 "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" <<EOF
UPDATE flashcards SET
  repetitions = 0,
  interval_days = 1,
  ease_factor = MAX(1.3, ease_factor - 0.2),
  next_review = datetime('now', '+1 day'),
  last_reviewed = datetime('now')
WHERE id = <card_id>;
EOF
```

### Step 4: Continue or finish

After each card, move to the next due card. When all due cards are done, show a summary:

```
Quiz complete!
- Cards reviewed: 5
- Got it: 3
- Missed: 2
- Next card due: tomorrow
```

### Important Rules

- **One card at a time.** Don't batch-show cards.
- **Always show the teaching note** when revealing the answer — it provides context for why this was a gap.
- **Don't re-quiz missed cards immediately** — they'll come back tomorrow (interval reset to 1 day).
- **Be encouraging.** Missing a card is normal and expected — that's why spaced repetition works.
````

- [ ] **Step 3: Verify the skill file was created**

Run: `ls -la "C:/Users/rberg/.claude/skills/forge-quiz/SKILL.md"`
Expected: File exists.

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/rberg/Documents/projects/ozian/forge"
git add -A
git commit -m "feat: create /forge-quiz skill with SM-2 spaced repetition"
```

---

### Task 4: Create the `/forge-review` Skill

**Files:**
- Create: `C:/Users/rberg/.claude/skills/forge-review/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p "C:/Users/rberg/.claude/skills/forge-review"
```

- [ ] **Step 2: Write the `/forge-review` SKILL.md**

Create `C:/Users/rberg/.claude/skills/forge-review/SKILL.md` with this content:

````markdown
---
name: forge-review
description: >
  Browse Forge learning progress. View flashcards, gaps, scaffolds, and sitting
  logs across assignments.
  Use when: "forge review", "show progress", "learning progress", "review flashcards",
  "show gaps", "what have I learned"
---

# Forge Review — Browse Your Learning

**Base directory:** `C:/Users/rberg/Documents/projects/ozian/forge`
**Database:** `C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db`

## Routing

Parse the user's request to determine what to show. If unclear, show the overview dashboard.

### Overview Dashboard (default)

Show a summary of all learning activity:

1. List assignments and their progress (same as `/forge status`)
2. Show total flashcard stats:
   ```bash
   sqlite3 -header -column "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
     "SELECT COUNT(*) as total_cards, SUM(CASE WHEN next_review <= datetime('now') THEN 1 ELSE 0 END) as due_now, ROUND(AVG(ease_factor), 2) as avg_ease, ROUND(AVG(repetitions), 1) as avg_reps FROM flashcards;"
   ```
3. Show upcoming review schedule:
   ```bash
   sqlite3 -header -column "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
     "SELECT DATE(next_review) as date, COUNT(*) as cards_due FROM flashcards WHERE next_review > datetime('now') GROUP BY DATE(next_review) ORDER BY date LIMIT 7;"
   ```

### Cards by Assignment

If the user asks about a specific assignment:

```bash
sqlite3 -header -column "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
  "SELECT id, concept, bloom_level, question, interval_days, next_review FROM flashcards WHERE assignment = '<slug>' ORDER BY bloom_level, concept;"
```

### Cards by Bloom's Level

```bash
sqlite3 -header -column "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
  "SELECT bloom_level, COUNT(*) as count, ROUND(AVG(ease_factor), 2) as avg_ease FROM flashcards GROUP BY bloom_level ORDER BY CASE bloom_level WHEN 'remember' THEN 1 WHEN 'understand' THEN 2 WHEN 'apply' THEN 3 WHEN 'analyze' THEN 4 WHEN 'evaluate' THEN 5 WHEN 'create' THEN 6 END;"
```

### Gap History

Read and display `assignments/<slug>/gaps.md` for a specific assignment.

### Sitting Logs

List and read sitting files from `assignments/<slug>/sittings/`.

### Card Detail

If the user asks about a specific card (by ID):

```bash
sqlite3 -header -column "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
  "SELECT * FROM flashcards WHERE id = <id>;"
```

## Important Rules

- **Read-only.** This skill never modifies data — it only reads and presents.
- **Format output clearly.** Use tables and markdown to make data easy to scan.
- **Offer navigation.** After showing any view, suggest what else they might want to see.
````

- [ ] **Step 3: Verify the skill file was created**

Run: `ls -la "C:/Users/rberg/.claude/skills/forge-review/SKILL.md"`
Expected: File exists.

- [ ] **Step 4: Commit**

```bash
cd "C:/Users/rberg/Documents/projects/ozian/forge"
git add -A
git commit -m "feat: create /forge-review skill for browsing learning progress"
```

---

### Task 5: Set Up the Daily Quiz Scheduled Task

**Files:**
- None (uses Claude Code scheduled task API)

- [ ] **Step 1: Create the scheduled task**

Use the `mcp__scheduled-tasks__create_scheduled_task` tool to create a daily morning quiz:

- **taskId:** `forge-daily-quiz`
- **cronExpression:** `0 9 * * 1-5` (weekdays at 9am local time)
- **prompt:** "Run /forge-quiz to quiz me on due flashcards. Check the database at C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db for cards where next_review <= now. If no cards are due, just let me know when the next one is due."
- **description:** "Daily Forge flashcard quiz — spaced repetition review each weekday morning"

- [ ] **Step 2: Verify the task was created**

Use `mcp__scheduled-tasks__list_scheduled_tasks` to confirm the task appears.

- [ ] **Step 3: Commit a note about the scheduled task**

Add a note to the project so future sessions know the scheduled task exists:

Write `C:/Users/rberg/Documents/projects/ozian/forge/CLAUDE.md`:

```markdown
# Forge — Personal Knowledge System

## Skills

- `/forge` — Assignment intake + Socratic dialogue (skill at `~/.claude/skills/forge/`)
- `/forge-quiz` — Daily spaced repetition quiz (skill at `~/.claude/skills/forge-quiz/`)
- `/forge-review` — Browse learning progress (skill at `~/.claude/skills/forge-review/`)

## Scheduled Tasks

- `forge-daily-quiz` — Runs `/forge-quiz` weekdays at 9am

## Data

- Assignments: `assignments/<slug>/`
- Flashcard DB: `flashcards/forge.db`
```

```bash
cd "C:/Users/rberg/Documents/projects/ozian/forge"
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md with skill and scheduled task reference"
```

---

### Task 6: End-to-End Smoke Test

**Files:** None (manual verification)

- [ ] **Step 1: Test `/forge` intake**

Invoke `/forge` and provide a simple text assignment like:
"Write a Python function that reads a CSV file and returns the average of a numeric column."

Verify:
- Assignment folder was created with a slug
- `assignment.md` contains the text
- `scaffold.md` was generated with Bloom's levels
- `gaps.md` was created (empty template)
- `sittings/` directory exists

- [ ] **Step 2: Test Socratic dialogue**

Continue the session after intake. Answer one concept correctly and one incorrectly. Verify:
- Scaffold updated with `understood` and `gap` statuses
- Gap recorded in `gaps.md` with teaching note
- Flashcard inserted into SQLite:
  ```bash
  sqlite3 -header -column "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" "SELECT * FROM flashcards;"
  ```
- Sitting log created in `sittings/`

- [ ] **Step 3: Test `/forge-quiz`**

Invoke `/forge-quiz`. Verify:
- Due cards are pulled and presented
- After grading, intervals update correctly:
  ```bash
  sqlite3 -header -column "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
    "SELECT id, concept, interval_days, ease_factor, repetitions, next_review FROM flashcards;"
  ```

- [ ] **Step 4: Test `/forge-review`**

Invoke `/forge-review`. Verify:
- Dashboard shows assignment progress and card stats
- Can drill into specific assignment cards

- [ ] **Step 5: Test `/forge status`**

Invoke `/forge status`. Verify:
- Shows progress table with correct counts

- [ ] **Step 6: Test `/forge continue`**

Start a new session. Invoke `/forge continue`. Verify:
- Lists assignments with unassessed concepts
- Resumes from where the last session left off
