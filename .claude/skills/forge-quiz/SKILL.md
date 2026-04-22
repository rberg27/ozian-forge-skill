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

Order: **unseen first** (reps = 0), **then struggled** (low `ease_factor`, reps > 0), **then easy/mastered** (ease_factor = 2.5, reps > 0). Within each bucket, lower ease and earlier `next_review` come first.

```bash
sqlite3 -header -column "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" \
  "SELECT id, assignment, concept, bloom_level, question, repetitions, ease_factor FROM flashcards WHERE next_review <= datetime('now') ORDER BY CASE WHEN repetitions = 0 THEN 0 WHEN repetitions > 0 AND ease_factor < 2.5 THEN 1 ELSE 2 END, ease_factor ASC, next_review ASC;"
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
   - If the user responds with an actual answer (not just "ready" / "show me"), treat that as their attempt — proceed to step 3.
   - If the user just says they're ready without giving an answer, proceed to step 3 without an attempt to assess.
3. When the user is ready, show the **answer** and the **teaching note** (if any)
4. **If the user gave an answer attempt in step 2**, analyze it against the reference answer and suggest a grade on the 4-level scale:
   - **easy** — nailed it fluently, no hesitation, captured all key points. Long interval.
   - **good** — correct with minor omissions or slight imprecision. Standard interval.
   - **hard** — got the gist but struggled, missed a meaningful piece, or needed to work for it. Short interval.
   - **missed** — wrong, blanked, or fundamentally off. Reset to 1 day.
   - Format: "My read: **[grade]** — [specific reasoning]. Recommend next review in ~N days."
   - Compute N using the formulas in Step 3 so the user sees the consequence of each grade.
   - Be honest and specific. Don't be generous just to be encouraging.
5. Ask: "Your call? (easy / good / hard / missed)" — **the user's self-grade is final**, even if it disagrees with your suggestion. Respect their call without pushing back.

### Step 3: Update the card based on grade

The grade determines the next interval and the change to the ease factor. Four grades:

| Grade   | New interval (reps=0) | New interval (reps=1) | New interval (reps≥2)          | Ease factor Δ    | Reps |
|---------|------------------------|------------------------|--------------------------------|------------------|------|
| easy    | 3 days                 | 10 days                | `interval * ease_factor * 1.3` | +0.15 (cap 2.5)  | +1   |
| good    | 1 day                  | 6 days                 | `interval * ease_factor`       | +0.10 (cap 2.5)  | +1   |
| hard    | 1 day                  | 3 days                 | `interval * 1.2`               | -0.15 (floor 1.3)| +1   |
| missed  | 1 day                  | 1 day                  | 1 day                          | -0.20 (floor 1.3)| → 0  |

**For "easy":**

```bash
sqlite3 "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" <<EOF
UPDATE flashcards SET
  repetitions = repetitions + 1,
  interval_days = CASE
    WHEN repetitions = 0 THEN 3
    WHEN repetitions = 1 THEN 10
    ELSE CAST(interval_days * ease_factor * 1.3 AS INTEGER)
  END,
  ease_factor = MIN(2.5, ease_factor + 0.15),
  next_review = datetime('now', '+' || CASE
    WHEN repetitions = 0 THEN '3'
    WHEN repetitions = 1 THEN '10'
    ELSE CAST(CAST(interval_days * ease_factor * 1.3 AS INTEGER) AS TEXT)
  END || ' days'),
  last_reviewed = datetime('now')
WHERE id = <card_id>;
EOF
```

**For "good":**

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

**For "hard":**

```bash
sqlite3 "C:/Users/rberg/Documents/projects/ozian/forge/flashcards/forge.db" <<EOF
UPDATE flashcards SET
  repetitions = repetitions + 1,
  interval_days = CASE
    WHEN repetitions = 0 THEN 1
    WHEN repetitions = 1 THEN 3
    ELSE MAX(1, CAST(interval_days * 1.2 AS INTEGER))
  END,
  ease_factor = MAX(1.3, ease_factor - 0.15),
  next_review = datetime('now', '+' || CASE
    WHEN repetitions = 0 THEN '1'
    WHEN repetitions = 1 THEN '3'
    ELSE CAST(MAX(1, CAST(interval_days * 1.2 AS INTEGER)) AS TEXT)
  END || ' days'),
  last_reviewed = datetime('now')
WHERE id = <card_id>;
EOF
```

**For "missed":**

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
