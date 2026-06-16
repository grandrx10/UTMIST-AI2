# AI Cubed (AI³) — UTMIST AI² 2025 Tournament Agent 🥉

[![Watch the Demo](https://img.youtube.com/vi/mZT_RHlAO7U/hqdefault.jpg)](https://www.youtube.com/watch?v=mZT_RHlAO7U)

[▶️ Match Demo (YouTube)](https://www.youtube.com/watch?v=mZT_RHlAO7U) · [📓 Colab Notebook](https://colab.research.google.com/drive/1V184vtHSagN13L0SbWGmnY-jCDvIefmm?usp=sharing) · [📄 Paper](https://drive.google.com/file/d/1G0hatGPBXvh2j5byjfrqKBthknNOt5sp/view) · [🏟️ Tournament Repo](https://utmist.gitlab.io/)

> A hand-crafted, rules-based agent for the [UTMIST](https://utmist.gitlab.io/) **AI² (AI Squared) 2025** 1v1 platform-fighting tournament — and it placed **3rd out of 400+ participants** in an event built around reinforcement learning, using no learning at all.

---

## 🏆 Result

**3rd Place — UTMIST AI² 2025 Tournament** (400+ participants)

The twist worth noting: AI² is a *reinforcement learning* tournament, where most submissions trained RL agents with the provided Stable-Baselines3 framework. AI Cubed took the opposite bet — a fully deterministic, if-statement agent driven by a hand-authored move database — and beat the large majority of trained models to reach the podium.

---

## Overview

**AI Cubed** is a competitive agent for the AI² environment: a Brawlhalla-inspired 1v1 platform fighter built in Pygame + PyMunk, with a custom multi-agent framework on top of Stable-Baselines3. Agents fight to knock each other off-stage while managing health, weapons, positioning, and recovery.

Instead of training a policy, AI Cubed is a **reactive, per-frame controller**: a small phase-based state machine decides *what to do* (survive / arm up / fight), and a weapon-aware move database decides *which attack to throw* given the opponent's exact position. It implements `SubmittedAgent(Agent)`, and the tournament server calls its `predict(obs)` once per frame.

---

## How It Works

### 1. The move database (the data step)

The core of the agent is a hand-built table characterizing every move of every weapon. For each of the three weapons — **Fist (0)**, **Spear (1)**, and **Hammer (2)** — I catalogued each attack by:

- **`keys`** — the input(s) that trigger it (e.g. `["s", "k"]`)
- **`type`** — whether it's a `ground` or `aerial` move
- **`range`** — its effective reach bucket: `close`, `far`, or `lunge`
- **`cover`** — the set of directional **sectors** the move actually hits (see below)
- **`super_safe`** — a flag for risky moves (e.g. the Spear's downward spike) that should only be thrown when firmly on-stage

Each weapon also carries a small **reach bonus** (`weapons_range`): Fist `0`, Spear `+0.3`, Hammer `+0.3`. This database is what lets the agent answer "given where the opponent is *right now*, which of my moves can reach them?" instead of guessing.

| Weapon | Reach bonus | Profile |
|--------|:-----------:|---------|
| **Fist (0)** | `0` | Default unarmed kit; broad mix of close / far / lunge options on both ground and air |
| **Spear (1)** | `+0.3` | Longest poke range; wide aerial coverage; includes a `super_safe` downward spike |
| **Hammer (2)** | `+0.3` | Heavy, high-commitment swings and big lunges |

### 2. The 8-sector targeting system

Each frame, the agent classifies the opponent's position into one of **8 directional sectors** relative to itself (above, below, left, right, and the four diagonals; `0` if overlapping), using a small dead-zone (`leeway = 0.3`) so near-aligned positions snap cleanly. Because the move table is authored for one facing direction, the sector is **mirrored horizontally when facing left** — so the same data works both ways.

### 3. Move selection — filter, then randomize

When in range and ready to attack, the agent:

1. Buckets the distance to the opponent into `close` / `far` / `lunge` / `out_of_range` (including the weapon's reach bonus).
2. Filters the current weapon's move list down to moves that match the **attack state** (ground vs aerial), the **range bucket**, the opponent's **sector**, and the `super_safe` constraint.
3. Picks **randomly** among all moves that pass the filter.

The randomness is intentional: among moves that are *all* valid for the situation, choosing unpredictably makes the agent much harder for an opponent to read and counter.

### 4. Opponent prediction (leading the target)

Before computing distance and sector, the agent extrapolates the opponent's position one frame ahead using their velocity (`opp_pos += opp_vel * 1/30`), so attacks are aimed at where the opponent is *going*, not where they were.

### 5. Phase-based state machine

A lightweight phase variable governs high-level intent:

- **`aggressive`** *(default)* — chase the opponent and attack when in range. Taking damage snaps the agent back into this phase.
- **`weapon_grab`** — when unarmed, route to the nearest active weapon spawner and pick it up (`h`) once within `0.5` units.
- **`flee`** — when unarmed *and* the opponent is armed *and* no spawner is reachable, retreat to a different platform rather than fight at a disadvantage.

The effective priority, in order: **stay alive → get a weapon → engage the enemy.**

### 6. Survival: recovery & spot dodge

- **Off-stage recovery** — when not safely on a platform, the agent finds the nearest platform (vertical distance weighted ×1.8 so it favors reachable ledges), moves toward its edge, jumps on a cooldown, and — if it's out of jumps in the air — fires the dedicated recovery move (`w` + `k`).
- **Reactive spot dodge** — if the opponent is within `1` unit **and** in `AttackState` (and a 50-frame cooldown has elapsed), the agent drops all other inputs and dodges (`l`) to slip the hit.

### Decision flow

```text
every frame:
    predict opponent position one frame ahead (lead the target)

    if opponent within 1 unit AND opponent is attacking (cooldown ok):
        → spot dodge                       # reactive defense, overrides all

    set phase:
        unarmed + spawner reachable        → weapon_grab
        unarmed + opponent armed + no spawner → flee
        otherwise / armed / took damage    → aggressive

    if not safely on a platform:
        → recover to nearest platform (jump / recovery move)

    elif phase == aggressive and in range:
        bucket distance → close / far / lunge
        filter weapon moves by (ground|aerial, range, opponent sector, super_safe)
        → pick a valid move at random

    else:
        → move toward target (opponent, spawner, or flee platform)
```

---

## Why a rules-based agent instead of RL?

In a tournament explicitly designed around reinforcement learning, a deterministic agent is a deliberate, contrarian choice — and the 3rd-place finish is the argument for it:

- **Predictable and debuggable** — every decision traces to an explicit rule, so failures are diagnosable instead of opaque.
- **No training instability or reward-shaping headaches** — the agent's quality is bounded by the move database and the phase logic, both of which I controlled directly.
- **Survival-first beats raw aggression** — recovery, fleeing when out-gunned, and reactive dodging make the agent hard to knock out, which wins a game decided by knockouts.
- **Domain knowledge as data** — encoding *how each move behaves* into a structured table turned out to be a faster path to strong play than learning it from scratch.

---

## Demo

[![AI Cubed match demo](https://img.youtube.com/vi/mZT_RHlAO7U/hqdefault.jpg)](https://www.youtube.com/watch?v=mZT_RHlAO7U)

▶️ **[Watch a full tournament match on YouTube](https://www.youtube.com/watch?v=mZT_RHlAO7U)**

<!-- Optional: keep the in-repo gif if you have it -->
<!-- <img src="docs/aisquaredv4.gif"></img> -->

---

## Running the Agent

The agent is submitted as the `SubmittedAgent(Agent)` class. The tournament server instantiates it and calls `predict(obs)` each frame. To run it locally against the provided baselines (`SB3Agent`, `BasedAgent`):

> 📝 **TODO — confirm/adjust to match the official harness's actual entry point.**

```bash
# 1. Set up the AI² environment (see the tournament repo)
pip install -r requirements.txt

# 2. Drop SubmittedAgent into the submission cell / file and run a match
#    against one of the provided baseline agents.
```

This agent is purely rules-based, so it needs **no model download** — the `_gdown` / `learn` / `save` hooks are inherited scaffolding from the RL template and aren't used at inference time.

---

## Tech Stack

- **Python** — all agent logic
- **NumPy** — observation handling
- **Pygame + PyMunk** — game environment and physics (provided by AI²)
- **Stable-Baselines3 / sb3-contrib** — the tournament's RL framework (imported by the template; AI Cubed itself uses none of it at runtime)

---

## Limitations & Future Work

- **One-frame lookahead only.** Beyond leading the opponent by a single frame, there's no multi-step planning, so the agent can be baited by setups a few frames ahead.
- **Hand-tuned thresholds.** Distance buckets, dead-zones, and cooldowns are set manually; a small search could sharpen them.
- **Random move choice is unweighted.** Among valid moves it picks uniformly; weighting by damage / safety / KO potential would likely raise its ceiling.
- **No opponent modeling.** It reacts to the current frame but doesn't adapt to a specific opponent's tendencies over a match.
- **Possible next step:** use the same move database to *shape or seed* an RL policy — combining the reliability of the rules with learned long-term play.

---

## Team

**Team AI Cubed (AI³)** — one better than AI Squared. 😄

> 📝 **TODO — add team members** (names / GitHub / roles).

---

## Acknowledgments

Built for the **[UTMIST](https://utmist.gitlab.io/) AI² 2025 Tournament**. Thanks to the UTMIST team for the environment, framework, and event.

---

## License

> 📝 **TODO — add a license** (e.g. MIT) if you want this to be reusable.
