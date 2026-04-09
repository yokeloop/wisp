# wisp — roadmap (draft)

three releases. each is independently shippable and usable.

---

## MVP — a real tamagotchi, no LLM required

**goal:** the core care loop works. a unique pet is born, persists, decays, and responds to care. no Ollama dependency — epics 1–3 run entirely offline.

**ships when:** `wisp status` shows a named pet with decaying stats after a session gap, and `wisp feed` visibly changes them.

### epic 1 · identity & persistence

- `WispState` TypeScript schema (all fields, types, defaults)
- hatch logic: species, name, rarity, shiny drawn from weighted random seeded by `hostname`
- save / load via `Bun.file('~/.wisp/state.json')`
- `wisp status` renders ASCII sprite + stats block via `clack.note`
- `bun link` global install

**done when:** running `wisp` twice shows the same named pet with lower stats on the second run.

### epic 2 · care actions

- commands: `feed`, `feed --snack`, `play`, `sleep`, `clean`, `heal`, `pet`
- stat delta table applied and clamped on each action
- `computeMood` from avg stats, displayed in status
- `clack.intro` / `clack.outro` wrapping each session
- clack spinner on file save

**done when:** `wisp feed` then `wisp status` shows hunger increased, cleanliness slightly down.

### epic 3 · decay & growth

- `applyDecay` on startup using `lastSeen` elapsed hours
- health penalty when hunger < 10 for extended time
- life stage transitions: egg → baby → child → teen → adult
- trait assignment at child stage from care pattern counters
- adult form determination at transition (playful / pristine / radiant / feral)
- age shown in status
- sprite variants per stage (minimum: egg, baby, adult × 3 moods)

**done when:** a pet neglected overnight has visibly lower stats; a 10-day-old well-cared pet shows adult sprite.

---

## V1 — the pet feels alive

**goal:** LLM dialogue works. the pet talks back in character. dev workflow integration is active. wisp is a daily companion.

**ships when:** `wisp talk` returns an in-character reply and `git commit` triggers a one-line wisp reaction.

### epic 4 · LLM dialogue

- `buildSystemPrompt` from WispState — ≤80 tokens
- `callOllama` with `gemma3:1b` default, `qwen3:0.6b` fallback
- `wisp talk <message>` with clack spinner during inference
- rolling 5-message in-session history (not persisted)
- trait influence baked into prompt phrasing
- graceful fallback to static mood reply if Ollama not running

**done when:** `wisp talk "how are you"` returns a 1–2 sentence reply that changes based on current mood.

### epic 5 · dev integration

- `wisp review` — runs `git diff HEAD~1`, passes to LLM with wisp personality
- `wisp hook install` — writes post-commit hook to `.git/hooks/`
- `wisp hook uninstall` — removes it cleanly
- post-commit: applies stat delta, calls Ollama for one-line commit reaction, prints to stdout
- idle nudge: if `lastSeen` > 4h on startup, wisp notes it has been waiting

**done when:** making a commit prints a one-line in-character reaction without any manual command.

### epic 6 · extended care

- `heal` command (health recovery)
- `clean` command (cleanliness)
- `pet` command (pure happiness, no stat cost)
- `feed --snack` variant with health trade-off
- shiny sprite variant for the 1/256 cases
- random event on startup (5% chance): illness, mood spike, found item

**done when:** all 9 care commands work; rare events appear occasionally on startup.

---

## V2 — long-term motivation

**goal:** there is a reason to come back tomorrow and next month. progression, collection, and social mechanics.

**ships when:** `wisp badges` shows earnable milestones and a second run on a fresh machine produces a different species.

### epic 7 · progression

- XP system: every care action and dev event awards XP
- 10 milestone badges with unlock conditions:
  - first feed, first commit, first talk, 7-day streak, 30-day streak,
    first evolution, adult reached, legendary rarity, shiny hatched, all-100 stats
- `wisp badges` command — shows earned / locked badges
- friendship bond score (0–100) derived from XP and care consistency, shown in status

**done when:** `wisp badges` lists at least 5 earned badges after a week of use.

### epic 8 · collection & social

- species collection log — tracks which species have been hatched across resets
- `wisp export` — prints an ASCII pet card to stdout (copyable for sharing)
- `wisp reset` — starts over with confirmation prompt, adds current species to collection log
- `wisp pause` / `wisp resume` — freezes decay (vacation mode)

**done when:** `wisp export` produces a shareable card; `wisp reset` preserves collection history.

### epic 9 · polish

- `wisp --help` and per-command help text
- full sprite set: all 18 species × 5 stages × 3 mood variants
- adult form sprites: 4 variants per species
- `WISP_MODEL` env var for custom model override
- `WISP_DIR` env var for custom state directory
- demo GIF in README
- seasonal skin flag (disabled by default, feature-flagged)

**done when:** a new user can clone, run `bun link`, and have a working named pet in under 2 minutes.

---

## release summary

| release | weeks | key milestone                                        |
| ------- | ----- | ---------------------------------------------------- |
| MVP     | 1–3   | `wisp status` with decay, care loop, growth — no LLM |
| V1      | 4–6   | `wisp talk` + git hook — pet is a daily companion    |
| V2      | 7–10  | badges, collection, export — long-term motivation    |

epics within each release can be parallelised. cross-release dependencies: V1 requires epic 1–2 complete. V2 requires all V1 epics.
