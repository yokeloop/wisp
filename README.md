# wisp (draft)

**a local LLM pet that lives in your terminal**

wisp is a tamagotchi powered entirely by your machine. it runs on [Ollama](https://ollama.com) with small language models — no cloud, no account, no data leaving your computer. your wisp has a name, a species, a mood, and a personality shaped by how you treat it.

```
$ wisp status

  ╭──────────────────────────────╮
  │  ≽^•⩊•^≼  Mochi             │
  │  Rare · Arctic Fox · day 4   │
  │                              │
  │  hunger      ████░░░   61   │
  │  happiness   █████░░   78   │
  │  energy      ███░░░░   44   │
  │  health      ██████░   91   │
  │                              │
  │  mood: tired                 │
  │  "could really use a nap"    │
  ╰──────────────────────────────╯
```

---

## what it is

- **a real pet** — hatches once, persists between sessions, ages over days
- **locally intelligent** — powered by `gemma3:1b` or `qwen3:0.6b` via Ollama, ~2GB RAM, works on CPU
- **personality-driven** — traits like `curious`, `snarky`, or `lazy` shape how it talks
- **care loop** — feed it, play with it, let it sleep; ignore it and stats decay
- **grows** — egg → baby → child → teen → adult; adult form reflects quality of care
- **dev-aware** — reacts to git commits, comments on your diffs via `wisp review`

---

## requirements

- [Bun](https://bun.sh) 1.x
- [Ollama](https://ollama.com) running locally

```bash
ollama pull gemma3:1b   # ~2GB, recommended
# or
ollama pull qwen3:0.6b  # ~1GB, low-spec hardware
```

## install

```bash
git clone https://github.com/your-org/wisp
cd wisp
bun install
bun link
```

## commands

```bash
wisp              # same as wisp status
wisp status       # show pet, stats, mood
wisp feed         # feed a meal (+hunger)
wisp feed --snack # quick snack (+mood, -health)
wisp play         # play together (+happiness, -energy)
wisp sleep        # rest (+energy)
wisp clean        # clean up (+cleanliness)
wisp heal         # treat illness (+health)
wisp pet          # just pet it (+happiness)
wisp talk "hey"   # chat — LLM responds in character
wisp review       # wisp comments on your last git diff
wisp hook install # add post-commit hook to current repo
wisp badges       # show earned milestones
```

---

## how it works

stats are pure numbers managed by TypeScript. the LLM never modifies state — it only generates personality-driven text from a compact system prompt built from current stats:

```
WispState (numbers) → system prompt → Ollama → text reply
```

decay is calculated from `lastSeen` timestamp on each startup. no background process, no daemon.

state is stored in `~/.wisp/state.json`.

---

## license

MIT
