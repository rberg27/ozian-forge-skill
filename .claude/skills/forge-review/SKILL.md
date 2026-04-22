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
