# wisp — blueprint (draft)

technical specification: what exists, how it works, and why decisions were made this way.

---

## decisions log

decisions that are not obvious and should not be revisited without reason.

| decision       | choice                      | rationale                                                                                                                                          |
| -------------- | --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| runtime        | Bun                         | native TypeScript execution, no build step, fast startup (<10ms), built-in `fetch` and file APIs — no `node-fetch`, no `fs` boilerplate            |
| CLI UX         | @clack/prompts              | `intro`/`outro`/`spinner`/`note` compose into a terminal experience that feels intentional, not printf                                             |
| persistence    | JSON (`~/.wisp/state.json`) | zero dependencies, human-readable, trivially portable; SQLite adds complexity with no benefit at this data scale                                   |
| LLM runtime    | Ollama                      | local inference, simple REST API on `localhost:11434`, no auth, works offline                                                                      |
| default model  | `gemma3:1b` QAT 4-bit       | 128K context window, ~2GB RAM, runs on CPU — best capability/weight ratio for personality dialogue on constrained hardware                         |
| fallback model | `qwen3:0.6b`                | ~1GB RAM for machines with less than 4GB available                                                                                                 |
| LLM role       | text-only                   | the model never reads or writes numbers — it receives a pre-built system prompt and returns a personality reply. all stat logic is pure TypeScript |
| decay strategy | on-startup calculation      | no background process or daemon needed; elapsed time from `lastSeen` is sufficient for a tamagotchi-scale simulation                               |
| install        | `bun link`                  | global `wisp` command without npm publish; one command, works immediately                                                                          |

---

## file structure

```
wisp/
├── package.json              # "bin": { "wisp": "./src/index.ts" }
├── tsconfig.json
├── src/
│   ├── index.ts              # #!/usr/bin/env bun — entry point, CLI routing
│   ├── commands/
│   │   ├── status.ts         # wisp status
│   │   ├── talk.ts           # wisp talk <message>
│   │   ├── feed.ts           # wisp feed [--snack]
│   │   ├── play.ts           # wisp play
│   │   ├── sleep.ts          # wisp sleep
│   │   ├── clean.ts          # wisp clean
│   │   ├── heal.ts           # wisp heal
│   │   ├── pet.ts            # wisp pet
│   │   ├── review.ts         # wisp review (git diff → LLM)
│   │   ├── badges.ts         # wisp badges
│   │   └── hook.ts           # wisp hook install | uninstall
│   ├── engine/
│   │   ├── state.ts          # WispState type, load/save, hatch, computeMood
│   │   ├── stats.ts          # applyDecay, applyAction, clamp
│   │   ├── llm.ts            # buildSystemPrompt, callOllama
│   │   └── growth.ts         # life stage transitions, evolution rules, trait assignment
│   └── renderer/
│       ├── index.ts          # renderStatus() — clack.note wrapper
│       └── sprites.ts        # ASCII art: Record<Species, Record<Stage, string>>
└── ~/.wisp/
    └── state.json            # persisted WispState (created on first run)
```

sprites are stored as a plain TypeScript object — `sprites[species][stage]` returns an ASCII string. no external files, no assets directory.

---

## CLI routing

entry point `src/index.ts`:

```typescript
#!/usr/bin/env bun
import { status } from "./commands/status";
import { talk } from "./commands/talk";
import { feed } from "./commands/feed";
import { play } from "./commands/play";
import { sleep } from "./commands/sleep";
import { clean } from "./commands/clean";
import { heal } from "./commands/heal";
import { pet } from "./commands/pet";
import { review } from "./commands/review";
import { badges } from "./commands/badges";
import { hook } from "./commands/hook";

const [, , cmd, ...args] = Bun.argv;

switch (cmd) {
  case "status":
    await status();
    break;
  case "talk":
    await talk(args.join(" "));
    break;
  case "feed":
    await feed(args);
    break;
  case "play":
    await play();
    break;
  case "sleep":
    await sleep();
    break;
  case "clean":
    await clean();
    break;
  case "heal":
    await heal();
    break;
  case "pet":
    await pet();
    break;
  case "review":
    await review();
    break;
  case "badges":
    await badges();
    break;
  case "hook":
    await hook(args[0]);
    break;
  default:
    await status();
    break;
}
```

all commands follow the same contract: `load state → mutate → save state → render`.

---

## WispState schema

```typescript
type Species =
  | "arctic-fox"
  | "axolotl"
  | "bat"
  | "capybara"
  | "cat"
  | "crow"
  | "dragon"
  | "ferret"
  | "firefly"
  | "fox"
  | "frog"
  | "jellyfish"
  | "moth"
  | "otter"
  | "owl"
  | "raccoon"
  | "salamander"
  | "wolf";

type Rarity = "common" | "uncommon" | "rare" | "epic" | "legendary";
type Stage = "egg" | "baby" | "child" | "teen" | "adult";
type Trait = "curious" | "lazy" | "snarky" | "kind" | "chaotic";
type AdultForm = "playful" | "pristine" | "radiant" | "feral";

interface WispState {
  // identity — set at hatch, never changes
  id: string; // sha256(hostname + bornAt)[:8]
  name: string; // generated from species-themed word list
  species: Species;
  rarity: Rarity;
  isShiny: boolean; // 1/256 chance at hatch
  bornAt: string; // ISO timestamp

  // stats — 0 to 100, managed by engine/stats.ts only
  hunger: number;
  happiness: number;
  energy: number;
  health: number;
  cleanliness: number;

  // growth
  stage: Stage;
  adultForm: AdultForm | null; // set on adult transition
  traits: Trait[]; // assigned at child stage, 1–2 traits
  age: number; // days since bornAt (computed on load)
  xp: number;

  // time
  lastSeen: string; // ISO timestamp — written on every save
}
```

`mood` is **never stored** — computed on the fly:

```typescript
export function computeMood(s: WispState): string {
  const avg = (s.hunger + s.happiness + s.energy + s.health) / 4;
  if (avg > 80) return "cheerful";
  if (avg > 60) return "content";
  if (avg > 40) return "tired";
  if (avg > 20) return "grumpy";
  return "miserable";
}
```

rarity is drawn at hatch using weighted random:

```typescript
const RARITY_WEIGHTS = {
  common: 55,
  uncommon: 25,
  rare: 13,
  epic: 6,
  legendary: 1,
};
```

---

## decay formula

calculated on every startup from elapsed time since `lastSeen`. no cron, no daemon.

```typescript
export function applyDecay(state: WispState): WispState {
  const hours = (Date.now() - new Date(state.lastSeen).getTime()) / 3_600_000;
  const factor = Math.min(hours, 12); // cap at 12 hours — prevents overnight kills

  return {
    ...state,
    hunger: clamp(state.hunger - factor * 3.5),
    happiness: clamp(state.happiness - factor * 2.0),
    energy: clamp(state.energy - factor * 1.5),
    cleanliness: clamp(state.cleanliness - factor * 1.0),
    // health only drops if hunger hits 0 for extended periods — see applyHungerPenalty
  };
}

const clamp = (n: number) => Math.max(0, Math.min(100, Math.round(n)));
```

health decay is separate — triggered only when `hunger < 10` for more than 6 elapsed hours.

---

## stat delta table

all values are additive deltas. result is clamped to 0–100 after application.

| command        | hunger | happiness | energy | health | cleanliness |
| -------------- | ------ | --------- | ------ | ------ | ----------- |
| `feed`         | +25    | +5        | 0      | 0      | −5          |
| `feed --snack` | +10    | +15       | 0      | −5     | −3          |
| `play`         | −10    | +20       | −15    | 0      | −5          |
| `sleep`        | 0      | +5        | +40    | +5     | 0           |
| `clean`        | 0      | +8        | 0      | +5     | +35         |
| `heal`         | 0      | 0         | 0      | +30    | 0           |
| `pet`          | 0      | +10       | 0      | 0      | 0           |
| `git commit`   | 0      | +8        | −3     | 0      | 0           |

---

## LLM prompt strategy

the model receives a compact system prompt assembled from current state. total target: ≤80 tokens (critical for 1B models — long prompts cause context loss and personality drift).

```typescript
export function buildSystemPrompt(s: WispState): string {
  const mood = computeMood(s);
  const traitLine = s.traits.length ? `Traits: ${s.traits.join(", ")}.` : "";

  return `You are ${s.name}, a ${s.rarity} ${s.species}.
Stage: ${s.stage}. ${traitLine}
Mood: ${mood}. Energy ${s.energy}/100, happiness ${s.happiness}/100.
Reply in 1–2 sentences. Stay in character. Never mention numbers or stats.`;
}
```

trait influence on tone — applied through prompt phrasing, not separate instructions:

| trait     | effect on reply                            |
| --------- | ------------------------------------------ |
| `curious` | asks a follow-up question, notices details |
| `lazy`    | short replies, mentions tiredness          |
| `snarky`  | mild sarcasm, dry humor                    |
| `kind`    | warm, supportive, uses "we"                |
| `chaotic` | topic jumps, unexpected associations       |

Ollama call — streaming disabled for CLI simplicity, 5-message rolling history (in-memory only, not persisted):

```typescript
export async function callOllama(
  system: string,
  history: { role: "user" | "assistant"; content: string }[],
): Promise<string> {
  const res = await fetch("http://localhost:11434/api/chat", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: Bun.env.WISP_MODEL ?? "gemma3:1b",
      stream: false,
      messages: [{ role: "system", content: system }, ...history],
    }),
  });
  if (!res.ok) throw new Error(`Ollama error: ${res.status}`);
  const data = await res.json();
  return data.message.content as string;
}
```

if Ollama is not running, commands that require LLM (`talk`, `review`) fall back to a mood-based static reply from a small lookup table — no crash.

---

## growth rules

### stage transitions

| from → to    | condition                             |
| ------------ | ------------------------------------- |
| egg → baby   | age ≥ 0.04 days (≈1 hour after hatch) |
| baby → child | age ≥ 2 days AND avg stats ≥ 45       |
| child → teen | age ≥ 5 days AND avg stats ≥ 50       |
| teen → adult | age ≥ 10 days AND avg stats ≥ 55      |

### trait assignment (at child stage)

traits are derived from the dominant care pattern up to that point, tracked as care event counters in state:

- most actions were `feed` → `lazy`
- most actions were `play` → `curious`
- low avg happiness throughout → `snarky`
- consistently high all stats → `kind`
- stats variance high (spiky care) → `chaotic`

### adult form (at adult transition)

determined by stat profile at time of transition:

| condition                        | adult form |
| -------------------------------- | ---------- |
| happiness > 70 AND health < 60   | `playful`  |
| health > 80 AND cleanliness > 75 | `pristine` |
| all stats ≥ 65                   | `radiant`  |
| any stat ≤ 20                    | `feral`    |
| otherwise                        | `playful`  |

---

## git hook integration

`wisp hook install` writes to `.git/hooks/post-commit` in the current repo:

```bash
#!/bin/sh
wisp hook commit
```

`wisp hook commit` (called by the hook):

1. loads WispState
2. applies commit delta (`happiness +8, energy −3`)
3. runs `git log -1 --pretty=%s` to get commit message
4. calls Ollama with a short prompt: react to this commit message in character
5. prints one line to stdout via `clack.log.info`
6. saves state

output example:

```
  ≽^•⩊•^≼  "a refactor? bold move. i believe in you."
```
