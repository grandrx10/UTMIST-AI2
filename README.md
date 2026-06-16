# AI Cubed (AI³) — UTMIST AI² 2025 Tournament Agent 🥉

[![Watch the Demo](https://img.youtube.com/vi/mZT_RHlAO7U/hqdefault.jpg)](https://www.youtube.com/watch?v=mZT_RHlAO7U)

[▶️ Match Demo (YouTube)](https://www.youtube.com/watch?v=mZT_RHlAO7U) · [📓 Colab Notebook](https://colab.research.google.com/drive/1V184vtHSagN13L0SbWGmnY-jCDvIefmm?usp=sharing) · [📄 Paper](https://drive.google.com/file/d/1G0hatGPBXvh2j5byjfrqKBthknNOt5sp/view) · [🏟️ Tournament Repo](https://utmist.gitlab.io/)

> A hand-crafted, frame-by-frame decision agent for the [UTMIST](https://utmist.gitlab.io/) **AI² (AI Squared) 2025** 1v1 platform-fighting tournament — and it placed **3rd out of 400+ participants** in an event built around reinforcement learning, using no learning at all.

---

## 🏆 Result

**3rd Place — UTMIST AI² 2025 Tournament** (400+ participants)

The twist worth noting: AI² is a *reinforcement learning* tournament, where most submissions trained RL agents with the provided SB3 framework. AI Cubed took the opposite bet — a fully deterministic, rules-based agent driven by empirical move data — and beat the large majority of trained models to reach the podium.

---

## Overview

**AI Cubed** is a competitive agent for the AI² environment: a Brawlhalla-inspired 1v1 platform fighter built in Pygame + PyMunk, with a custom multi-agent RL framework on top of Stable-Baselines3. Agents fight to knock each other off-stage while managing health, weapons, positioning, and recovery.

Instead of training a policy, AI Cubed plays as a **greedy, per-frame state machine**: every single frame it evaluates the current game state and the set of available actions, then commits to the move with the best immediate payoff under a fixed priority hierarchy.

---

## How It Works

### 1. Move characterization (the data step)

Before writing any logic, I profiled every move available in the environment — measuring how each one actually behaves in practice (startup, active frames, damage, range, recovery/end-lag, and the situations where it wins). This empirical profile is what the agent's decisions are built on: rather than guessing which attack is "good," it picks the move whose measured properties best fit the current frame.

> 📝 **TODO — drop in your real numbers.** This table is the centerpiece of the "I gathered data" story, so fill it with the actual values you measured. Even rough buckets (fast / medium / slow) are compelling.

| Move | Startup | Damage | Range | Recovery | Best Used When |
|------|:-------:|:------:|:-----:|:--------:|----------------|
| _e.g. Light attack_ | _fast_ | _low_ | _short_ | _short_ | _enemy adjacent, safe poke_ |
| _e.g. Heavy attack_ | _slow_ | _high_ | _medium_ | _long_ | _enemy committed / stunned_ |
| _Dodge / spot dodge_ | — | — | — | _short_ | _enemy close and attacking_ |
| _Weapon attack_ | _…_ | _…_ | _…_ | _…_ | _… _ |
| _…_ | | | | | |

### 2. Greedy per-frame decision making

Each frame is treated independently. The agent reads the state (its own position/health, the enemy's position/health and action, weapon availability, stage boundaries) and selects the single best move for *that* frame — no planning ahead, no learned value function, just the locally optimal choice given the priority order below.

### 3. Priority hierarchy

The agent resolves what to do in a strict order of importance:

1. **Survive first** — self-preservation overrides everything. Recover to the stage when off-ledge, avoid getting hit, and don't take risks that could lead to a knockout.
2. **Control weapons second** — once safe, prioritize acquiring and using weapons for the advantage they provide.
3. **Engage the enemy last** — only after survival and weapon priorities are satisfied does it close distance and attack.

### 4. Reactive spot dodging

Layered on top: if the enemy is **close** *and* **currently attacking**, the agent performs a **spot dodge** to slip the incoming hit and create a punish window — a reactive override that protects the survival priority above.

### Decision flow

```text
every frame:
    if enemy is close AND enemy is attacking:
        → spot dodge                       # reactive defense
    elif self is in danger:                # off-stage, about to be hit, low margin
        → prioritize survival              # recover / retreat / defend
    elif a weapon is available:
        → secure or use the weapon         # positional advantage
    else:
        → close distance and choose the    # greedy: best move by measured data
          optimal attack for this frame
```

---

## Why a state machine instead of RL?

In a tournament explicitly designed around reinforcement learning, a deterministic agent is a deliberate, contrarian design choice — and the 3rd-place finish is the argument for it:

- **Predictable and debuggable** — every decision traces back to an explicit rule, so failures are diagnosable instead of opaque.
- **No training instability or reward-shaping headaches** — the agent's quality is bounded by the move data and the priority logic, both of which I controlled directly.
- **Survival-first beats aggression** — many trained agents over-commit to attacking; prioritizing not-dying turns out to be a strong baseline in a knockout-based game.

---

## Demo

[![AI Cubed match demo](https://img.youtube.com/vi/mZT_RHlAO7U/hqdefault.jpg)](https://www.youtube.com/watch?v=mZT_RHlAO7U)

▶️ **[Watch a full tournament match on YouTube](https://www.youtube.com/watch?v=mZT_RHlAO7U)**

<!-- Optional: keep the in-repo gif if you have it -->
<!-- <img src="docs/aisquaredv4.gif"></img> -->

---

## Running the Agent

> 📝 **TODO — fill in your actual setup/run steps.** Example scaffold below.

```bash
# 1. Clone and install
git clone <your-repo-url>
cd <your-repo>
pip install -r requirements.txt

# 2. Run AI Cubed against a baseline / another agent
python run_match.py --agent ai_cubed --opponent <opponent>
```

The agent entry point lives in `<path/to/agent_file.py>` and exposes the action-selection function the tournament server calls each frame.

---

## Tech Stack

- **Python** — agent logic
- **Pygame + PyMunk** — game environment and physics (provided by AI²)
- **Stable-Baselines3** — the tournament's multi-agent RL framework (used by the environment; AI Cubed itself is rules-based)

---

## Limitations & Future Work

> 📝 Optional, but this section reads well — it shows you understand the tradeoffs you made.

- **No long-horizon planning.** Greedy per-frame choices can be baited by opponents who set traps a few frames ahead.
- **Hand-tuned priorities.** Thresholds (what counts as "close," "in danger") are manually set; a small search or learned tuning could sharpen them.
- **Static to opponent behavior.** The agent doesn't adapt mid-match to a specific opponent's tendencies.
- **Possible next step:** use the same move-characterization data to seed or shape an RL policy — combining the reliability of the rules with learned long-term play.

---

## Team

**Team AI Cubed (AI³)** — a step up from AI Squared. 😄

> 📝 **TODO — add team members** (names / GitHub / roles).

---

## Acknowledgments

Built for the **[UTMIST](https://utmist.gitlab.io/) AI² 2025 Tournament**. Thanks to the UTMIST team for the environment, framework, and event.

---

## License

> 📝 **TODO — add a license** (e.g. MIT) if you want this to be reusable.
