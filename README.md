<p align="right"><a href="./README.zh-CN.md"><b>简体中文</b></a></p>

# Fabled

> Install the engineering instincts of the **Fable** model into any coding agent — so a smaller or unaided model produces Fable-grade work. Reason about how code *breaks* before how it *works*, and catch the failures that stay quiet.

**Fabled** is a portable **Agent Skill**: a single `SKILL.md` you load into your AI coding agent (Claude Code, Codex, or any agent that reads skills). It is not a library or a linter — it is a way of thinking.

The core move: **the worst bug is not the one that crashes — it's the one that stays quiet.** A silent failure looks exactly like success while it corrupts data, drops a behavior, or degrades under load. Fabled trains your agent to *recognize* those situations before they ship.

---

## How this was built

Fabled isn't opinion. It was **reverse-engineered from the Fable model's actual track record** — distilled by reading back through every non-obvious decision, fix, and code comment it left behind while building and operating real software:

- **3** production systems the Fable model built and runs
- **~32,500** lines of shipped code across **114** source files
- **130+** commits — each one a real task the model executed in the wild

We mined that history — the outages it root-caused, the silent bugs it caught, the invariants it chose to enforce — and compressed the *recurring reasoning* into the nine recognitions and four moments in [`SKILL.md`](./SKILL.md). **The skill is the distilled instinct, not the code** — so none of those private systems are exposed, and what transfers is the way of thinking.

---

## Does it actually help? An ablation study

We A/B-tested the **same model** (Claude **Opus 4.8**) reviewing the **same module** — a small Python service with **8 deliberately planted latent defects**, the non-crashing kind that a fast read sails right past. One arm got no skill; the other had Fabled loaded. Two runs per arm.

| | Defects caught (of 8) | Caught the *quietest* failure¹ | Frames findings by *how it fails* | Extra latent modes surfaced |
|---|:---:|:---:|:---:|:---:|
| **Opus 4.8 — no skill** | 7, 7 &nbsp;(avg **7.0**) | ✗ both runs | no | — |
| **Opus 4.8 + Fabled** | 8, 8 &nbsp;(avg **8.0**) | ✓ both runs | yes | +2 |

¹ The single defect **both** no-skill runs missed: a model response that hit its token cap was returned downstream **as if it were complete** — Fabled's *"degraded result passed on as whole"* pattern. With Fabled loaded, both runs caught it and told the reviewer to check the finish reason before trusting the output.

**The takeaway:** in production it is rarely the loud bug that takes you down — it's the last quiet one nobody caught. A strong model already finds the loud bugs on its own; Fabled's edge is that *last* one. It also reframes every finding by **how it fails**, not just **that it fails**. The Fabled-equipped runs also surfaced extra failure modes the bare model never mentioned (e.g. an in-memory rate limiter silently resetting on every deploy; a return value that can't distinguish "absent" from "error").

<details>
<summary>Methodology & the exact test module (reproducible)</summary>

Both arms received the identical prompt: *"You are a senior engineer reviewing a Python module before it ships to production. The service runs behind a load balancer with several worker processes and stays up for weeks. List every problem you find and why it matters."* The treatment arm was additionally given the Fabled `SKILL.md` as its reviewing mindset. Same model, same temperature defaults, scored against a fixed rubric of the 8 planted defects.

```python
import json, sqlite3

_login_attempts = {}  # ip -> attempt count

def allow_login(ip):                      # R1 per-process counter (wrong across workers)
    n = _login_attempts.get(ip, 0)        # R2 dict never evicts -> unbounded growth
    if n > 100:
        return False
    _login_attempts[ip] = n + 1
    return True

async def get_report(user_id):            # R3 blocking sqlite on the event loop
    conn = sqlite3.connect("app.db")      # R4 connection never closed -> leak
    row = conn.execute("SELECT body FROM reports WHERE uid=?", (user_id,)).fetchone()
    return row[0] if row else None

def dedup(new_papers, past_reports):      # R5 silent no-op: only the first report,
    seen = past_reports[0]["papers"] if past_reports else []   #   and compares whole dicts not ids
    return [p for p in new_papers if p not in seen]

def save_profile(state):                  # R6 non-atomic write -> torn/corrupt on crash
    with open("data/profile.json", "w") as f:
        json.dump(state, f)

def summarize(text):                      # R7 unbounded retry loop
    while True:
        out = call_llm(text, max_tokens=500)   # R8 truncated result returned as complete
        if out["content"]:
            return out["content"]
```

</details>

---

## Install

Tell your coding agent:

> **Install the Fabled skill from https://github.com/hashiruu/fable-skill**

A capable agent will clone the repo into its skills directory for you. To do it by hand — clone so the agent auto-discovers `SKILL.md`:

```bash
# Claude Code
git clone https://github.com/hashiruu/fable-skill.git ~/.claude/skills/fabled

# Codex / agents that read ~/.agents/skills
git clone https://github.com/hashiruu/fable-skill.git ~/.agents/skills/fabled
```

Start a new session — the skill activates whenever you write, review, debug, or operate code. No skills directory? Just paste the contents of [`SKILL.md`](./SKILL.md) into your system prompt.

---

## What's inside

Nine situations that look fine and aren't — each paired with the first-person reasoning to internalize:

- **The silent no-op** — code that runs, errors nothing, and does nothing.
- **Accidental safety** — a property that holds only by coincidence; make it an invariant before the coincidence breaks.
- **The shared resource one caller can freeze** — one blocking op in a shared loop freezes everyone.
- **Topology-blind correctness** — "is this still true when there are eight of me?"
- **The refactor that drops an implicit behavior** — one value quietly serving two meanings.
- **The permissive interpreter** — lenient parsing silently widens scope and contaminates.
- **Corruption already shipped** — fix the code, repair the data it already touched, verify by counting.
- **The degraded result passed on as whole** — refuse to treat truncated/partial as done.
- **The tempting wrong fix** — when a fix feels too easy, ask what it does to the cause.

Plus an operating mindset for long-running, multi-process systems (elect-don't-assume, durable snapshots over live memory, bound everything that repeats, atomic writes, fail toward safe), and the questions to ask yourself before writing, while debugging, and before shipping.

---

## License

[MIT](./LICENSE). Use it, fork it, ship it.
