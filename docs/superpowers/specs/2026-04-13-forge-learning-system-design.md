# Forge — Personal Knowledge System

**Date:** 2026-04-13
**Status:** Design

## Overview

Forge is a terminal-based personal knowledge system built as a set of Claude Code skills. It takes assignments (text or PDF), scaffolds the required concepts using Bloom's taxonomy, runs Socratic dialogue to identify knowledge gaps, teaches missing concepts in the moment, and creates targeted flashcards for long-term retention via spaced repetition.

## Folder Structure

```
forge/
├── assignments/
│   ├── <assignment-slug>/
│   │   ├── assignment.md          # Original assignment (text or extracted PDF)
│   │   ├── scaffold.md            # Bloom's taxonomy breakdown of skills/concepts
│   │   ├── gaps.md                # Identified knowledge gaps with teaching notes
│   │   └── sittings/
│   │       ├── 2026-04-13.md      # Dialogue log from each sitting
│   │       └── 2026-04-15.md
│   └── <another-assignment>/
│       └── ...
├── flashcards/
│   └── forge.db                   # SQLite — all flashcards across assignments
└── docs/
    └── ...
```

- `forge.db` is a single database across all assignments. Each card links back to its source assignment via an `assignment` column.
- `scaffold.md` breaks down required skills by Bloom's level and tracks assessment status.
- `gaps.md` records what was missing and what was taught.
- Sittings are dated dialogue logs for reviewing progression.

## Skills

### `/forge` — Main Entry Point

Subcommands:

- `/forge` or `/forge new` — Start a new assignment intake
- `/forge continue` — Resume Socratic dialogue on current/specified assignment
- `/forge status` — Show progress across assignments (concepts assessed, gaps found, cards created)

### `/forge-quiz` — Daily Quiz

- Pulls all cards where `next_review <= today` from SQLite
- Presents cards in order (oldest due first)
- Phase 1 (current): Show question, user thinks, reveal answer, user self-grades (got it / didn't)
- Phase 2 (future): Each card becomes a mini Socratic dialogue — answer is evaluated, follow-ups probe deeper, new gaps can spawn new cards
- Updates spaced repetition intervals based on grade

### `/forge-review` — Browse Progress

- View flashcards by assignment, by Bloom's level, or by due date
- See gap history and teaching notes
- Browse sitting logs

### Scheduled Task

A daily scheduled task (via Claude Code scheduled tasks) fires each morning and runs `/forge-quiz`, pulling due cards and quizzing the user.

## Assignment Intake Flow

1. User invokes `/forge` or `/forge new`
2. Skill asks for the assignment — user pastes text or points to a file/PDF in the `forge/` directory
3. System generates an assignment slug (e.g. `rest-api-design`) and creates the subfolder
4. Original content saved as `assignment.md` (text saved directly; PDF content extracted and saved)
5. System analyzes the assignment and produces `scaffold.md`

### Scaffold Structure (scaffold.md)

Each concept/skill required by the assignment, organized by Bloom's level:

- **Remember:** Key terms, definitions, facts
- **Understand:** Explain concepts in own words
- **Apply:** Use the concept to solve a problem
- **Analyze:** Break down relationships between concepts
- **Evaluate:** Judge approaches, trade-offs
- **Create:** Synthesize into something new

Each item in the scaffold has a status: `not assessed`, `gap`, `understood`.

## Socratic Dialogue Flow

The core learning loop, invoked during `/forge new` (after intake) and `/forge continue`:

1. **Pick next concept** — Select the next unassessed concept from the scaffold. Start at the bottom of Bloom's (Remember) and work up. No point probing higher levels if basics aren't solid.

2. **Probe** — Ask the user to explain the concept. Open-ended, not multiple choice. e.g. "Can you explain what idempotency means in the context of REST APIs?"

3. **Socratic dig** — Evaluate the response against Bloom's criteria for that level:
   - **Remember:** Did they state the fact/definition accurately?
   - **Understand:** Did they explain in their own words with correct reasoning?
   - **Apply:** Can they walk through a concrete example?
   - **Analyze/Evaluate/Create:** Higher-order probing as appropriate

   If the answer is partial or off, ask targeted follow-up questions (2-3 rounds) to map the boundary of what the user actually knows vs. doesn't. The goal is to pinpoint the specific gap, not just detect that one exists.

4. **Teach** — Once the specific gap is clear, explain that nugget. Teaching is targeted because the Socratic phase already narrowed down exactly what's missing. Not a lecture — a precise explanation of the specific misunderstanding or missing knowledge.

5. **Verify** — Quick check that the teaching landed. e.g. "So in your own words, what's the difference between X and Y?"

6. **Record outcome:**
   - Understood → mark as `understood` in scaffold
   - Gap found and taught → mark as `gap` in scaffold, add to `gaps.md` with the teaching note, create flashcard(s) in SQLite

7. **Log dialogue** — Append the full exchange to `sittings/YYYY-MM-DD.md` (if multiple sittings on the same day, append to the same file with a timestamp separator)

8. **Repeat** — Move to the next concept

The user can stop a session at any time. Progress is saved in the scaffold and sitting logs so `/forge continue` picks up where they left off.

## Flashcard System

### SQLite Schema

```sql
CREATE TABLE flashcards (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    assignment    TEXT NOT NULL,     -- slug linking back to assignment folder
    concept       TEXT NOT NULL,     -- what skill/concept this tests
    bloom_level   TEXT NOT NULL,     -- remember/understand/apply/analyze/evaluate/create
    question      TEXT NOT NULL,     -- the quiz question
    answer        TEXT NOT NULL,     -- expected answer or key points
    teaching_note TEXT,              -- the explanation given during the session
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
    next_review   DATETIME NOT NULL, -- when this card is next due
    interval_days INTEGER DEFAULT 1, -- current spacing interval
    ease_factor   REAL DEFAULT 2.5,  -- SM-2 algorithm multiplier
    repetitions   INTEGER DEFAULT 0, -- consecutive correct answers
    last_reviewed DATETIME
);
```

### Card Types by Bloom's Level

- **Remember:** "What is X?" / "Define Y"
- **Understand:** "Explain in your own words why X happens"
- **Apply:** "Given this scenario, how would you use X?"
- **Analyze:** "What's the relationship between X and Y?"
- **Evaluate:** "What are the trade-offs between approach A and B?"
- **Create:** "How would you design a solution for X?"

### Spaced Repetition (SM-2 Algorithm)

- New cards: `interval_days = 1`, `ease_factor = 2.5`, `repetitions = 0`
- On correct answer:
  - If `repetitions == 0`: interval = 1
  - If `repetitions == 1`: interval = 6
  - If `repetitions >= 2`: interval = previous interval * ease_factor
  - ease_factor += 0.1 (capped at 2.5 total)
  - repetitions += 1
- On wrong answer:
  - interval = 1
  - repetitions = 0
  - ease_factor -= 0.2 (floor at 1.3)
- Card is due when `next_review <= today`

### Quiz Phases

**Phase 1 (passive, initial implementation):**
- Show question
- User thinks about the answer
- Reveal answer and teaching note
- User self-grades: "got it" or "didn't"
- Update intervals based on grade

**Phase 2 (interactive, future evolution):**
- Show question
- User provides their answer
- System evaluates the answer via Socratic follow-ups
- If correct: card progresses normally
- If gap found: mini teaching moment, potentially spawns new cards
- Update intervals based on outcome

## Design Principles

- **The system doesn't lecture — it discovers.** Socratic dialogue first, teaching only after the gap is precisely identified.
- **Flashcards capture the specific misunderstanding**, not generic definitions. A card born from "you confused PUT and PATCH" is more useful than "define PUT."
- **Domain-agnostic.** The system works with any subject matter — it scaffolds based on what the assignment requires, not a fixed curriculum.
- **Progressive depth.** Bloom's levels ensure basics are solid before probing higher-order thinking.
- **Everything is logged.** Sitting logs, gap records, and the scaffold together form a complete picture of the learning journey.
