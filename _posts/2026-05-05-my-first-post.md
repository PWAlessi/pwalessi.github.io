---
layout: post
title: "My First Post"
date: 2026-05-05
---
# IRONVEIL Devlog

A development log for IRONVEIL — a tactical squad wargame in pre-production. New posts at the top.

---

## #001 — From Design Document to Technical Spec
**2026-05-04 · Patrick**

Two weeks ago I had a design document. Fifty pages of pillars, tables, mechanics, and aspirations. It described a tactical wargame called IRONVEIL — squad-scale, WeGo turn structure, fire-superiority combat, LLM-driven enemy AI, procgen maps, named soldiers, all of that. It was well-developed as game design.

It was useless as engineering input.

Today I have a technical spec that says exactly what to build. This post is about the gap between those two documents and the decisions that filled it.

### The shape of the gap

A good design doc tells you what the player will experience. A good tech spec tells you what the program will do. The translation between them is where projects get killed. You can ship a game with mediocre design. You cannot ship one with mediocre technical foundations — those decide whether your game is debuggable, replayable, moddable, performant, and survivable through three years of changes.

For IRONVEIL the gap had specific shape. The design said things like:

- "Determinism: same inputs produce same outputs." Whose `Random()`? At what bit width? In which language? Across CPUs?
- "Hex grid, offset coordinates." Which offset variant? What does a hex object actually contain? How do you put a wall *between* two hexes?
- "LLM-driven AI." How fast? What schema? What happens when the model is offline?
- "ER formula = Base × Range × Cover × ..." What functions produce those modifiers? What numbers?

A few hundred questions like that. I worked through them by batching related ones together and forcing concrete decisions before moving on.

### Foundation: coordinates, determinism, language boundary

**Coordinates.** Axial `(q, r)` for storage, cube `(x, y, z)` for distance and line and rotation operations, never offset (except at the render boundary, where Godot's TileMap wants it). Offset coordinates are a tile-editor convenience that a procgen-first game doesn't need. Cube makes line-drawing trivial; axial is a clean two-int store.

**Determinism.** This is the one most people get wrong. "Same seed, same outcome" is not a property you get for free — it's a property you fight for. Floats are non-deterministic across CPUs. C#'s `Random` and Godot's `RandomNumberGenerator` are implementation-defined. Dictionary iteration order is undefined. Adding a new random call between version 0.4 and 0.5 silently breaks every old replay.

The contract I locked in:

- Custom PCG32, hand-implemented bit-exact in both C# and GDScript and validated against each other.
- Q16.16 fixed-point integers for **all** resolution math. No floats anywhere in the resolver.
- One master seed per turn, named substreams derived by hashing — `prng_for("casualty", turn)` — so adding a new substream later doesn't perturb existing replays.
- Stable sort by UUIDv4 string comparison for any "simultaneous" loop.

The fixed-point decision is the painful one. GDScript doesn't overload operators, so multiplication looks like `Q16.mul(a, b)` instead of `a * b`. It's ugly. It's also the only way I can guarantee a save made on someone's Ryzen reproduces correctly on someone else's Intel chip three years later. The ugliness is the price of cross-platform replay determinism, and that's a price I'm willing to pay because replay is also where my fine-tuning data, debugging, and (eventually) play-by-email all live.

**Language boundary.** GDScript owns canonical game state. C# is a stateless solver called once per turn with a JSON payload: `resolve_turn(state, orders, balance, seed) → (new_state, events)`. The C# resolver is a black-box function I can unit-test in isolation and reuse later for headless replay or PBEM.

There's a cost. Path and LOS queries during order-input UI hit JSON serialize on every cursor hover, which is too hot. So GDScript reimplements A* and LOS for **preview only** — the C# implementations remain canonical for resolution. Two implementations of the same algorithm sounds bad until you realize they catch each other's bugs in CI.

### Why the LLM is the doctrine engine, not the game AI

This took me a while to articulate well. The instinct in 2026 is "AI agent does AI things." That instinct is wrong for tactical wargames specifically.

LLMs are slow, variable, occasionally nonsensical. Resolution math needs to be fast and deterministic. Putting LLM output anywhere in the resolution math path is a bug. The LLM's job is **intent and expression** — what does the OPFOR squad try to do this turn, and what does a panicking veteran say when his stress crosses 0.5? Those are doctrine and personality questions. The math layer doesn't care about doctrine. It cares about ER values and seeded rolls.

So the architecture splits cleanly:

- **Tier 1, deterministic, always active:** pathfinding, LOS, suppression response, withdrawal trigger, casualty rolls. The simulation core.
- **Tier 2, optional enhancement:** LLM-driven intent declaration, chaos event selection, doctrine-consistent decision patterns, behavioral expression. Optional because the game must remain fully playable on machines without the LLM, or when the LLM call fails.

The transport: HTTP POST to localhost (Ollama in dev, llama.cpp server in ship), 3-second timeout, OpenAI-compatible chat-completions schema, JSON Schema validation on every response with one retry on schema failure, fallback to a deterministic behavior tree on second failure or timeout. **Single batched call per turn** — all unit intents and the chaos recommendation come back in one structured response. Mid-mission, if the model crashes, the game continues on the BT with a single "enhanced AI unavailable" toast. No mission abort.

The LLM call is also where the long-term play opens up. Every request and response is logged to disk regardless of build mode. Across thousands of playtests, that becomes a corpus of (game_state, intent, outcome) triples — fine-tuning data for a smaller, faster, IRONVEIL-specific model. The architecture supports that without retrofitting because the I/O capture is baked in from day one.

### Walls and hedges live on edges, not in hexes

A small structural decision that prevents a category of bugs.

The mistake I almost made: putting walls and hedges in the hex's `features[]` array along with buildings and craters. That's wrong. A wall sits *between* hexes, not inside one. A hex with a wall on its north side has cover from the north only. And a wall is **shared** between two hexes; if I store it in both, I get a sync bug the first time something forgets the mirror write.

The fix: half-edge canonical ownership. Each hex stores its 6 directional edges, but only owns the data for directions 0, 1, 2. Reading directions 3, 4, 5 redirects to the neighbor's edges 0, 1, 2 (rotated by `(d + 3) % 6`). One source of truth, no mirror writes, JSON round-trip safe.

I've been burned enough times by mirrored data that getting this right early is worth the slightly more complex lookup helper.

### Math: numbers in files, hash in saves

The combat formulas had ranges (`Cover_Modifier ∈ [0.3–1.0]`) but no functions. I locked them in:

- Cover values map to multipliers: 0→1.0, 1→0.8, 2→0.6, 3→0.4
- Stress bands map to morale modifiers: Normal 1.0, Shaken 0.85, Stressed 0.65, Breaking and Broken 0.50
- Experience modifier scales with squad-average XP: 0.70 at all-green, 1.30 at maxed
- Casualty probability multiplies base lethality (per weapon class), inverse cover, and suppression duration
- Detection probability is a squared falloff with distance, gated by concealment and signature modifiers
- Stress accumulates +0.05/turn under light suppression, +0.10 under heavy, +0.20 per KIA, decays −0.05 on Hold or −0.15 on Rally

Every one of these numbers lives in `/data/balance/*.json`. None hardcoded. The one decision that pays off everywhere: the C# resolver receives the entire balance struct in its per-turn JSON payload. C# never reads files. That means hot-reload during DEV "just works" without any C#-side reload logic, and replay determinism is preserved by hashing the balance struct into every save and event log entry.

`balance_hash` and a human-readable `balance_label` (e.g., `"v0.4-lethal-buffed"`) get embedded in saves and events. Replays refuse to play if the hash mismatches. This catches the silent-divergence case where someone tweaks a number and old replays "work" but produce different results.

Splitting balance into nine small JSON files instead of one mega-file is small but pays off in playtest velocity — a designer tuning detection ranges doesn't merge-conflict with one tuning casualty rates.

### Meta-progression: veterans you can lose

The original design was pure roguelike. Campaign ends, start over, nothing persists except your skill at the game. I pushed back on this. The "Named Soldiers, Persistent Stakes" pillar wants meta-progression that amplifies attachment, not the kind that gives you arcade-style perks.

Three layers:

1. **Veteran roster.** Up to 8 surviving soldiers bank between campaigns. Up to 6 can deploy in any future campaign. They keep names, specialties, and experience (capped at 3 on banking; further XP earned in new campaigns can push back up to 5). If a campaign fails, every veteran **deployed in that campaign** is lost permanently. The pool doesn't dilute permadeath — it amplifies it. The decision to bring your best soldiers on a hard mission becomes a real one.

2. **Doctrine unlocks.** Achievement-gated additions to the *menu of choices*, not raw power. "Win a Recon mission with zero casualties" unlocks Sniper Pair as an attachment option. "Win an Assault while flanking an objective" unlocks a small Bounding Doctrine bonus. New options, not new strength.

3. **Memorial.** Permanent records of KIA. Names, missions served, citations generated by the LLM post-mission. No mechanical effect. Pure narrative weight, but the data is cheap and the emotional payoff is large for the named-soldier pillar.

What I deliberately left out: a currency-based perk shop. That model fits *Hades* and *Dead Cells*. It does not fit a defense-credibility-positioned tactical wargame. Spending command points to buy +5% accuracy is the wrong vibe.

### Debugging is load-bearing infrastructure

Halfway through I got a directive that I should have led with: extensive debug features baked into every system from day one. Right. Solo dev on a complex emergent stack (WeGo + LLM + chaos + procgen + named-soldier persistence) cannot afford to bolt debug tooling on later.

What ships in the spec from day one:

- Per-turn event log entries for every meaningful state change, in JSON Lines format
- Toggleable in-game overlays for LOS, cover, concealment, detection range, full ER breakdown on hover, A* pathfinding visualization, edge features, squad facing arcs, procgen layer-by-layer step-through
- Inspector panels for any unit, soldier, hex, or engagement that show **all hidden fields** including the exact LLM input/output that affected them
- A debug console with commands to set state, force chaos events, jump turns, dump and load state, replay seeds, diff replays, hot-tweak balance values
- Always-on LLM I/O capture in both DEBUG and RELEASE builds (this is also the fine-tuning corpus)
- A standalone CLI determinism-diff tool that replays recorded turns; CI fails on any divergence
- Time-travel: the last 20 autosaves are retained per campaign, scrubbable backward, with the ability to modify orders and re-execute forward
- Crash dumps that capture full state, last 50 events, last 5 LLM exchanges, build info

Debug tooling is not polish. It is the difference between "I can ship this" and "I have no idea why this broke."

### What I'd do differently

The biggest lesson: I should have done this exercise before fully writing the design doc, or at least in conversation with it. The design doc gestured at things like "structured output" and "deterministic resolution" without committing to *how*. Once I was forced to commit, several design choices became cleaner — the morale modifier combined nicely once stress and cohesion had concrete bands; the chaos engine's pre-resolution vs post-resolution split fell out naturally once phase ordering was nailed down; the LLM context format simplified once I introduced auto-generated zones at procgen time.

The design doc isn't wasted — it's the *why*. But the why and the how need to be developed in conversation, not in sequence.

### What's next

Milestone 0: hex grid rendering, the GDScript↔C# boundary in its smallest possible form, and the first deterministic-replay smoke test. The smallest interesting piece I can build that exercises the foundation without committing to any gameplay yet. If the boundary marshaling is awkward or the determinism diff is slow, I want to find out before I've written ten thousand lines on top of it.

The next post will probably be about whether axial-vs-cube conversions cost what I think they cost, what JSON payload sizes actually look like across the boundary, and what failure modes show up when I run the deterministic replay test on the first real codebase instead of a paper spec.

— *Patrick*
