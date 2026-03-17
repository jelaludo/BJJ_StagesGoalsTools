# Factored Markov Visualization — Design Spec

**Date:** 2026-03-17
**Status:** Draft
**Author:** Gerald + Claude

## Summary

A single-page interactive HTML visualization that models BJJ positional control as parallel Markov chains. Each "stage" of a position (e.g., grip fight, hip control, head control, strangulation for back attacks) runs as an independent mini-chain with three states: controlled, contested, lost. The composite of all stage states determines the probability of finishing the opponent. A settings modal allows tuning simulation parameters.

This is a proof-of-concept. Architecture may change when integrated into a broader project.

## Context

Gerald has two existing static diagrams (screenshots) mapping BJJ positions as Stages → Goals → Tools/Options:

1. **Back Attacks** (source: J Danaher) — 4 control stages + 1 auxiliary bail-out
2. **Butterfly Attacks** (source: Marcelo Garcia) — 4 control stages + 1 auxiliary bail-out

These diagrams present positions as linear progressions, but in reality control is non-linear: you don't "win" a stage and keep it forever. Each stage is constantly contested. This project models that reality.

A related project (GameLoops/MarkovChains/MarkovChainViz.html) models grappling as a flat Markov chain with states like Disengaged → Engaged → Dominant → Sub Attempt → Tap. This project differs by running multiple chains in parallel — one per control dimension — rather than one chain with sequential states.

## Conceptual Model

### Factored Markov Chain

Each position (e.g., Back Attacks) has N stages. Each stage is a 3-state Markov chain running in parallel:

```
Controlled ⇄ Contested ⇄ Lost
```

Transitions are only between adjacent states (no skipping from Controlled directly to Lost).

The composite of all stage states produces a finish probability. Having all stages controlled makes finishing near-certain. Losing any stage degrades the probability. Below a threshold, the practitioner should bail to an auxiliary position.

### Cascading Dependencies

Stages are not fully independent. Losing control of an upstream stage raises the probability of downstream stages degrading. For back attacks:

- Grip Fight cascades to → Hip Control, Head Control
- Hip Control cascades to → Head Control, Strangulation
- Head Control cascades to → Strangulation

The cascade direction and strength are position-specific. Cascade strength is tunable via the settings modal.

### Stage 5: Auxiliary

Stage 5 is not a Markov chain. It is a decision point: when the composite finish probability drops below a configurable threshold, the practitioner should switch to an auxiliary technique (Kimura, Gift Wrap, Triangle, etc.) to gain tempo on the next position rather than persist in a losing back attack.

## Data Model

```js
const positions = {
  backAttacks: {
    title: "BJJ Back Attacks",
    subtitle: "V20190507 by Gerald",
    source: "J Danaher",
    stages: [
      {
        name: "Grip Fight",
        goal: "Advantageous grips, shoulder control, negate one hand",
        tools: [
          "Dynamic grips", "Cross-grip", "Always a threat",
          "Leg to block arm", "Weight on arm", "\"magic grip\""
        ],
        transitions: {
          controlled_to_contested: 0.30,
          contested_to_controlled: 0.50,
          contested_to_lost: 0.20,
          lost_to_contested: 0.40
        },
        cascadesTo: [1, 2]  // indices of Hip Control, Head Control
      },
      {
        name: "Hip Control",
        goal: "Control opponent's hips, don't let him cross your ankle/knee",
        tools: [
          "Dynamic hooks", "Switch left-right", "Body lock", "\"single back\""
        ],
        transitions: {
          controlled_to_contested: 0.25,
          contested_to_controlled: 0.45,
          contested_to_lost: 0.25,
          lost_to_contested: 0.35
        },
        cascadesTo: [2, 3]  // Head Control, Strangulation
      },
      {
        name: "Head Control",
        goal: "Control opponent's head between your head and strangle arm",
        tools: [
          "Use own head", "Hand on head", "Close the circle"
        ],
        transitions: {
          controlled_to_contested: 0.35,
          contested_to_controlled: 0.40,
          contested_to_lost: 0.25,
          lost_to_contested: 0.30
        },
        cascadesTo: [3]  // Strangulation
      },
      {
        name: "Strangulation",
        goal: "Sink it deep past the chin, tap/pass out",
        tools: [
          "Attack up-down", "Flat fist", "Switch left-right",
          "Finger crawl", "Mixed tempo", "Hide the hand"
        ],
        transitions: {
          controlled_to_contested: 0.20,
          contested_to_controlled: 0.35,
          contested_to_lost: 0.30,
          lost_to_contested: 0.25
        },
        cascadesTo: []
      }
    ],
    auxiliary: {
      name: "Switch to Auxiliary",
      description: "Accept back attack didn't work, gain tempo on next position",
      tools: [
        "Kimura grip", "Shoulder control", "Lockdown",
        "Gift Wrap", "Race to base", "Triangle"
      ],
      threshold: 0.25
    },
    cascadeStrength: 0.15,
    finishWeights: {
      controlled: 1.0,
      contested: 0.4,
      lost: 0.05
    }
  }
};
```

### Finish Probability Calculation

```
P(finish) = product of finishWeights[stage.state] for each stage, expressed as a percentage

Example:
  All controlled: 1.0 × 1.0 × 1.0 × 1.0 = 100%
  3 controlled, 1 contested: 1.0 × 1.0 × 1.0 × 0.4 = 40%
  2 controlled, 1 contested, 1 lost: 1.0 × 1.0 × 0.4 × 0.05 = 2%
```

### Cascade Mechanic

Cascading is **state-based** (continuous pressure, not event-based). Every tick, for each stage currently in "contested" or "lost", add `cascadeStrength` to the degradation probability of each stage in its `cascadesTo` list. The modifier applies to whichever degradation transition matches the target's current state:

- Target in "controlled" → modifier added to `controlled_to_contested`
- Target in "contested" → modifier added to `contested_to_lost`
- Target in "lost" → no effect (already fully degraded)

Multiple upstream stages stack additively. All modifiers reset and are recalculated at the start of each tick.

## Simulation Engine

### Tick Loop

Each tick (interval controlled by speed slider):

1. **Calculate cascade modifiers**: For each stage currently in "contested" or "lost", add `cascadeStrength` to the degradation probability of its `cascadesTo` targets (see Cascade Mechanic above for which probability is modified).
2. **Roll transitions**: For each stage, roll a single random number in [0, 1) against three mutually exclusive outcomes based on current state:
   - **If "controlled"**: `[0, p_degrade)` → move to contested; `[p_degrade, 1)` → stay. Where `p_degrade = controlled_to_contested + cascade_modifier`.
   - **If "contested"**: `[0, p_recover)` → move to controlled; `[p_recover, p_recover + p_degrade)` → move to lost; remainder → stay. Where `p_recover = contested_to_controlled`, `p_degrade = contested_to_lost + cascade_modifier`. All values clamped so the sum does not exceed 1.0.
   - **If "lost"**: `[0, p_recover)` → move to contested; remainder → stay. Where `p_recover = lost_to_contested`.
   Only one transition per stage per tick.
3. **Highlight tools**: For each stage that transitioned, randomly select 1-2 tools from that stage's tool list. These remain highlighted until the next transition in that stage.
4. **Recalculate finish probability** from the new state vector.
5. **Check terminal conditions**:
   - Finish prob ≥ 95% for 3 consecutive ticks → **Finish** (tap animation)
   - Finish prob ≤ auxiliary threshold → **Stage 5 activates** (bail animation)
6. **Log the tick** to the event log. When a cascade modifier contributed to a transition, note the source stage (e.g., "cascade from Grip Fight").

### Terminal Behavior

When a terminal condition fires (finish or bail), the simulation stops. A "Reset" button appears in the header. Clicking it restores all stages to "controlled" and clears the event log.

### Initial State

All stages start in "controlled" (attacker has just secured the back).

## UI Layout

### Structure

```
┌──────────────────────────────────────────────────────┐
│  BJJ Back Attacks                        ⚙  ▶⏸ ⏭  │
│  Source: J Danaher                                   │
├───────────┬───────────┬───────────┬──────────┬───────┤
│ 1. Grip   │ 2. Hip    │ 3. Head   │ 4.Strang │FINISH │
│ Fight     │ Control   │ Control   │ ulation  │       │
│           │           │           │          │       │
│ goal text │ goal text │ goal text │goal text │ bar   │
│           │           │           │          │ meter │
│ [CTRL]    │ [CTRL]    │ [CONT]    │ [LOST]   │       │
│  cont     │  cont     │  ctrl     │  cont    │ 35%   │
│  lost     │  lost     │  lost     │  ctrl    │       │
│           │           │           │          │       │
│ tools     │ tools     │ tools     │ tools    │       │
├───────────┴───────────┴───────────┴──────────┴───────┤
│ 5. Switch to Auxiliary                    [tools]    │
├──────────────────────────────────────────────────────┤
│ Event log (scrollable, newest first)                 │
└──────────────────────────────────────────────────────┘
```

### Visual Rules

- **Theme**: Dark background (#0c0c0c), monochrome UI, Georgia serif font
- **State colors** (applied only to the active state per column):
  - Controlled: muted green background (#1a2e1a), green text (#6abf6a)
  - Contested: neutral grey background (#1e1e1e), light grey text (#bbb)
  - Lost: muted red background (#2e1a1a), red text (#bf6a6a)
- **Inactive states**: dim (#151515 background, #444 text)
- **Active tool**: brighter text (#ccc), visible border (#555), subtle background (#222)
- **Inactive tool**: dim (#444 text, #282828 border)
- **Finish meter**: vertical bar, green-to-red gradient fill, dashed line at bail threshold
- **Stage 5 footer**: dashed border, dim by default, brightens when threshold crossed
- **Transitions**: CSS transitions on state and tool elements for smooth animation (~300ms)
- **Controls in header**: Gear icon (settings modal), Play/Pause toggle, Step button (advance one tick), Reset button (visible after terminal condition)
- **Each column shows all three state labels stacked vertically**. The active state gets the color accent; the other two are dim. This makes it easy to see where each stage sits in the controlled→contested→lost spectrum.

### Settings Modal (Gear Icon)

Overlay modal with sliders:

| Slider | Range | Default | Effect |
|--------|-------|---------|--------|
| Simulation Speed | 0.5x – 5x | 1x | Tick interval (1x = 1 second) |
| Attacker Skill | 1 – 10 | 5 | For each +1 above 5, add +0.03 to all `*_to_controlled` probs and subtract 0.03 from all `*_to_lost`/`*_to_contested` (from controlled) probs. Clamped to [0.05, 0.95]. |
| Defender Skill | 1 – 10 | 5 | For each +1 above 5, add +0.03 to all `*_to_lost`/`*_to_contested` (from controlled) probs and subtract 0.03 from all `*_to_controlled` probs. Clamped to [0.05, 0.95]. |
| Cascade Strength | 0 – 0.5 | 0.15 | How much upstream degradation affects downstream |
| Bail Threshold | 0.05 – 0.50 | 0.25 | Finish prob below this triggers Stage 5 |

Sliders update the simulation in real time (no need to restart).

### Event Log

Scrollable area at the bottom, newest entries first. Format:

```
[tick 23] Grip Fight: controlled → contested (using: Cross-grip)
[tick 23] Head Control: contested → controlled (using: Close the circle)
[tick 23] Finish probability: 40% → 28%
[tick 24] Hip Control: controlled → contested (cascade from Grip Fight)
```

## Technology

- Single HTML file, all CSS and JS inline
- No dependencies, no build step
- Position data as JS objects at the top of the file
- CSS transitions for state animations
- `setInterval` for simulation tick loop
- Same patterns as the existing MarkovChainViz.html

## Scope

### In (POC)

- Back Attacks position, fully wired with all 4 stages + auxiliary
- RNG simulation with play/pause/step
- Cascading dependencies between stages
- Settings modal with tunable sliders
- Tool highlighting during simulation
- Event log
- Finish/bail terminal conditions with simple animation

### Out (Future)

- Butterfly Attacks position (same structure, swap data object — add after POC validated)
- Position selector tabs (like MarkovChainViz.html tabs)
- Manual state override (clicking states to force transitions)
- Run history/statistics
- SVG node graph showing cascade connections
- Mobile-responsive layout

## Open Questions

1. **Transition probability tuning**: The base probabilities in the data model are educated guesses. They will need playtesting to feel realistic. The settings modal helps with this.
2. **Finish condition**: "95% for 3 consecutive ticks" is arbitrary. May need adjustment after seeing it run.
3. **Tool selection logic**: Currently random 1-2 tools per transition. Could be weighted or contextual (e.g., certain tools are more likely when recovering from "lost"). Keeping it random for POC.
