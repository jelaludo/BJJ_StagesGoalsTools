# Factored Markov Visualization — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-page HTML visualization of BJJ Back Attacks as 4 parallel Markov chains with cascading dependencies, RNG simulation, and tunable settings.

**Architecture:** Single monolithic HTML file. Position data as JS object at top, simulation engine in middle, DOM rendering at bottom. No dependencies, no build step. Patterns follow the existing MarkovChainViz.html from the GameLoops project.

**Tech Stack:** Vanilla HTML/CSS/JS, inline in one file. No frameworks.

**Spec:** `docs/superpowers/specs/2026-03-17-factored-markov-viz-design.md`

**Reference:** `G:/Dev/experiments/GameLoops/MarkovChains/MarkovChainViz.html` (existing Markov viz for style/pattern reference)

---

## File Structure

Single file:
- Create: `BackAttacksViz.html` — the entire POC

---

### Task 1: HTML Shell + CSS Theme

Create the HTML file with the complete CSS theme and static layout (no JS yet). This establishes the visual foundation that all subsequent tasks build on.

**Files:**
- Create: `BackAttacksViz.html`

- [ ] **Step 1: Create HTML with full CSS and static layout**

Write the complete `<!DOCTYPE html>` document with:
- All CSS in a `<style>` block (dark theme, monochrome, Georgia serif)
- Header bar with title "BJJ Back Attacks", subtitle "V20190507 by Gerald · Source: J Danaher", gear icon (⚙), play button (▶), step button (⏭), reset button (hidden by default, `display: none`)
- 4 stage columns + finish meter column in a flex container
- Each column: stage name, goal text, 3 state labels (Controlled/Contested/Lost) stacked vertically, tools section
- Finish meter: container div with inner fill div (height = finish prob %), green-to-red linear gradient, percentage text below, dashed threshold line positioned at 25% from bottom
- Stage 5 auxiliary footer (dashed border)
- Event log area (scrollable div)
- Settings modal (hidden by default, `display: none`) with 5 sliders: Speed, Attacker Skill, Defender Skill, Cascade Strength, Bail Threshold

**Required element IDs** (referenced by JS in later tasks):
- `#stages-container` — flex container for the 4 stage columns
- `#stage-0` through `#stage-3` — each stage column
- `#state-0-controlled`, `#state-0-contested`, `#state-0-lost` (and same for stages 1-3) — state label elements
- `#tools-0` through `#tools-3` — tools container per stage
- `#finish-bar` — inner fill div of finish meter
- `#finish-pct` — percentage text element
- `#finish-threshold` — dashed threshold line element
- `#auxiliary-footer` — Stage 5 footer
- `#event-log` — event log container
- `#btn-play`, `#btn-step`, `#btn-reset` — control buttons
- `#btn-gear` — gear icon
- `#settings-modal` — settings modal overlay

CSS values from spec:
- Background: `#0c0c0c`, font: `Georgia, serif`
- Active controlled: bg `#1a2e1a`, text `#6abf6a`
- Active contested: bg `#1e1e1e`, text `#bbb`
- Active lost: bg `#2e1a1a`, text `#bf6a6a`
- Inactive states: bg `#151515`, text `#444`
- Active tool: text `#ccc`, border `#555`, bg `#222`
- Inactive tool: text `#444`, border `#282828`
- Column borders: `#333`, column bg: `#111`
- All state/tool elements: `transition: all 0.3s`

Hardcode all 4 columns with their real stage names, goals, and tool lists from the spec data model. Hardcode initial state: all stages showing "Controlled" as active (green accent). Finish meter showing 100%.

- [ ] **Step 2: Verify in browser**

Open `BackAttacksViz.html` in a browser. Verify:
- Dark theme renders correctly
- 4 columns visible with correct stage names, goals, tools
- All stages show "Controlled" with green accent
- Finish meter shows 100%
- Stage 5 footer visible but dim
- Event log area visible but empty
- Gear icon visible, clicking it does nothing yet
- Settings modal is hidden

---

### Task 2: Position Data Object + DOM Rendering from Data

Replace the hardcoded HTML columns with JS that renders from the position data object. This decouples data from presentation so adding Butterfly later is just a data swap.

**Files:**
- Modify: `BackAttacksViz.html`

- [ ] **Step 1: Add the position data object**

Add a `<script>` block. Define the `positions.backAttacks` object exactly as specified in the spec (all 4 stages with names, goals, tools, transitions, cascadesTo, plus auxiliary, cascadeStrength, finishWeights).

Add a `state` object to track simulation state:

```js
const state = {
  stages: [], // will hold { current: 'controlled', activeTools: [] } per stage
  tickCount: 0,
  finishProb: 1.0,
  consecutiveFinishTicks: 0,
  running: false,
  terminated: false,
  intervalId: null,
  settings: {
    speed: 1,
    attackerSkill: 5,
    defenderSkill: 5,
    cascadeStrength: 0.15,
    bailThreshold: 0.25
  }
};
```

- [ ] **Step 2: Write the render function**

Write `function render(position, state)` that updates existing DOM elements in place (no rebuild):

```js
function render(position, state) {
  // Update each stage column
  for (let i = 0; i < position.stages.length; i++) {
    const stageState = state.stages[i];
    const states = ['controlled', 'contested', 'lost'];

    // Update state labels: active gets accent class, others get dim
    for (const s of states) {
      const el = document.getElementById(`state-${i}-${s}`);
      el.className = 'state-label' + (s === stageState.current ? ` state-active state-${s}` : ' state-dim');
    }

    // Update tools: active tools get highlight, others dim
    const toolsContainer = document.getElementById(`tools-${i}`);
    const toolEls = toolsContainer.querySelectorAll('.tool');
    toolEls.forEach(el => {
      const isActive = stageState.activeTools.includes(el.textContent);
      el.className = 'tool' + (isActive ? ' tool-active' : '');
    });
  }

  // Update finish meter
  const pct = state.finishProb * 100;
  document.getElementById('finish-bar').style.height = pct + '%';
  document.getElementById('finish-pct').textContent = pct.toFixed(0) + '%';

  // Update threshold line position
  const threshPct = state.settings.bailThreshold * 100;
  document.getElementById('finish-threshold').style.bottom = threshPct + '%';

  // Update Stage 5 footer
  const auxEl = document.getElementById('auxiliary-footer');
  auxEl.className = 'auxiliary-footer';
  if (state.terminated && state.finishProb <= state.settings.bailThreshold) {
    auxEl.classList.add('auxiliary-active');
  } else if (state.finishProb <= state.settings.bailThreshold * 1.5) {
    auxEl.classList.add('auxiliary-warning');
  }

  // Show/hide reset button
  document.getElementById('btn-reset').style.display = state.terminated ? 'inline-block' : 'none';

  // Terminal visual classes on finish meter
  const meterEl = document.getElementById('finish-bar').parentElement;
  meterEl.classList.remove('terminal-finish', 'terminal-bail');
  if (state.terminated) {
    meterEl.classList.add(state.finishProb >= 0.95 ? 'terminal-finish' : 'terminal-bail');
  }
}
```

Replace the hardcoded column content with static elements using the required IDs. Each column's tools section contains `<span class="tool">Tool Name</span>` elements.

- [ ] **Step 3: Write the init function**

Write `function init(position)` that:
1. Sets `state.stages` to an array of `{ current: 'controlled', activeTools: [] }` for each stage.
2. Calls `calculateFinishProb()` to set initial finish probability.
3. Calls `render()`.

Call `init(positions.backAttacks)` on page load.

- [ ] **Step 4: Write calculateFinishProb**

```js
function calculateFinishProb(position, state) {
  let prob = 1.0;
  for (const stage of state.stages) {
    prob *= position.finishWeights[stage.current];
  }
  state.finishProb = prob;
  return prob;
}
```

- [ ] **Step 5: Verify in browser**

Refresh the page. Should look identical to Task 1 — all controlled, 100% finish prob — but now rendered from data. Inspect the console to verify no errors.

---

### Task 3: Simulation Engine (Tick Loop)

Implement the core simulation: cascade calculation, transition rolls, tool highlighting, finish probability update, terminal conditions.

**Files:**
- Modify: `BackAttacksViz.html`

- [ ] **Step 1: Write getEffectiveTransitions**

```js
function getEffectiveTransitions(position, stageIndex, state) {
  const stage = position.stages[stageIndex];
  const base = { ...stage.transitions };
  const skillDelta = state.settings.attackerSkill - 5;
  const defDelta = state.settings.defenderSkill - 5;
  const skillMod = skillDelta * 0.03;
  const defMod = defDelta * 0.03;

  // Attacker skill: boost recovery, reduce degradation
  base.contested_to_controlled = Math.min(0.95, Math.max(0.05, base.contested_to_controlled + skillMod));
  base.lost_to_contested = Math.min(0.95, Math.max(0.05, base.lost_to_contested + skillMod));
  base.controlled_to_contested = Math.min(0.95, Math.max(0.05, base.controlled_to_contested - skillMod));
  base.contested_to_lost = Math.min(0.95, Math.max(0.05, base.contested_to_lost - skillMod));

  // Defender skill: boost degradation, reduce recovery
  base.controlled_to_contested = Math.min(0.95, Math.max(0.05, base.controlled_to_contested + defMod));
  base.contested_to_lost = Math.min(0.95, Math.max(0.05, base.contested_to_lost + defMod));
  base.contested_to_controlled = Math.min(0.95, Math.max(0.05, base.contested_to_controlled - defMod));
  base.lost_to_contested = Math.min(0.95, Math.max(0.05, base.lost_to_contested - defMod));

  // Cascade modifiers
  let cascadeMod = 0;
  for (let i = 0; i < position.stages.length; i++) {
    if (i === stageIndex) continue;
    const upstream = position.stages[i];
    const upstreamState = state.stages[i].current;
    if ((upstreamState === 'contested' || upstreamState === 'lost') &&
        upstream.cascadesTo.includes(stageIndex)) {
      cascadeMod += state.settings.cascadeStrength;
    }
  }

  const currentState = state.stages[stageIndex].current;
  if (currentState === 'controlled') {
    base.controlled_to_contested = Math.min(0.95, base.controlled_to_contested + cascadeMod);
  } else if (currentState === 'contested') {
    base.contested_to_lost = Math.min(0.95, base.contested_to_lost + cascadeMod);
  }

  return { transitions: base, cascadeMod };
}
```

- [ ] **Step 2: Write rollTransition**

```js
function rollTransition(position, stageIndex, state) {
  const { transitions, cascadeMod } = getEffectiveTransitions(position, stageIndex, state);
  const currentState = state.stages[stageIndex].current;
  const roll = Math.random();
  let newState = currentState;
  let cascadeSource = null;

  if (currentState === 'controlled') {
    if (roll < transitions.controlled_to_contested) newState = 'contested';
  } else if (currentState === 'contested') {
    // Spec: [0, pRecover) → controlled; [pRecover, pRecover+pDegrade) → lost; remainder → stay
    // Clamp pDegrade so total does not exceed 1.0 (preserves recovery probability)
    const pRecover = transitions.contested_to_controlled;
    const pDegrade = Math.min(transitions.contested_to_lost, 1.0 - pRecover);
    if (roll < pRecover) {
      newState = 'controlled';
    } else if (roll < pRecover + pDegrade) {
      newState = 'lost';
    }
  } else if (currentState === 'lost') {
    if (roll < transitions.lost_to_contested) newState = 'contested';
  }

  // Determine if cascade contributed to degradation
  if (cascadeMod > 0 && (
    (currentState === 'controlled' && newState === 'contested') ||
    (currentState === 'contested' && newState === 'lost')
  )) {
    // Find which upstream stage(s) caused the cascade
    for (let i = 0; i < position.stages.length; i++) {
      const upstream = position.stages[i];
      const upState = state.stages[i].current;
      if ((upState === 'contested' || upState === 'lost') &&
          upstream.cascadesTo.includes(stageIndex)) {
        cascadeSource = position.stages[i].name;
        break; // report first source
      }
    }
  }

  return { previousState: currentState, newState, cascadeSource };
}
```

- [ ] **Step 3: Write the tick function**

Add a stub at the top of the script block so terminal conditions don't throw before Task 4:
```js
function stopSimulation() {} // stub — replaced in Task 4
```

Then write the tick function:

```js
function tick(position, state) {
  if (state.terminated) return;
  state.tickCount++;
  const prevProb = state.finishProb;
  const logEntries = [];

  // Roll transitions for all stages
  for (let i = 0; i < position.stages.length; i++) {
    const result = rollTransition(position, i, state);
    if (result.newState !== result.previousState) {
      state.stages[i].current = result.newState;

      // Highlight 1-2 random tools
      const tools = position.stages[i].tools;
      const toolCount = Math.random() < 0.5 ? 1 : 2;
      const shuffled = [...tools].sort(() => Math.random() - 0.5);
      state.stages[i].activeTools = shuffled.slice(0, Math.min(toolCount, tools.length));

      let entry = `[tick ${state.tickCount}] ${position.stages[i].name}: ${result.previousState} → ${result.newState} (using: ${state.stages[i].activeTools.join(', ')})`;
      if (result.cascadeSource) {
        entry += ` [cascade from ${result.cascadeSource}]`;
      }
      logEntries.push(entry);
    }
  }

  // Recalculate finish probability
  calculateFinishProb(position, state);

  if (Math.abs(state.finishProb - prevProb) > 0.001) {
    logEntries.push(`[tick ${state.tickCount}] Finish probability: ${(prevProb * 100).toFixed(0)}% → ${(state.finishProb * 100).toFixed(0)}%`);
  }

  // Check terminal conditions
  if (state.finishProb >= 0.95) {
    state.consecutiveFinishTicks++;
    if (state.consecutiveFinishTicks >= 3) {
      state.terminated = true;
      stopSimulation();
      logEntries.push(`[tick ${state.tickCount}] ★ FINISH — Submission complete!`);
    }
  } else {
    state.consecutiveFinishTicks = 0;
  }

  if (!state.terminated && state.finishProb <= state.settings.bailThreshold) {
    state.terminated = true;
    stopSimulation();
    logEntries.push(`[tick ${state.tickCount}] ✕ BAIL — Switch to auxiliary (finish prob ${(state.finishProb * 100).toFixed(0)}% ≤ ${(state.settings.bailThreshold * 100).toFixed(0)}%)`);
  }

  appendLog(logEntries);
  render(position, state);
}
```

- [ ] **Step 4: Write appendLog helper**

```js
function appendLog(entries) {
  const logEl = document.getElementById('event-log');
  for (const entry of entries) {
    const div = document.createElement('div');
    div.className = 'log-entry';
    div.textContent = entry;
    logEl.insertBefore(div, logEl.firstChild);
  }
}
```

- [ ] **Step 5: Verify with manual tick**

Temporarily add a button or console call: `tick(positions.backAttacks, state)`. Open browser, click/call it several times. Verify:
- States change in the columns (green/grey/red accents shift)
- Tools highlight when a stage transitions
- Finish probability updates
- Event log shows entries with correct format
- Cascade attribution appears when relevant

Remove the temporary trigger after verifying.

---

### Task 4: Play/Pause/Step/Reset Controls

Wire up the header buttons to control the simulation.

**Files:**
- Modify: `BackAttacksViz.html`

- [ ] **Step 1: Write control functions**

```js
function startSimulation() {
  if (state.terminated || state.running) return;
  state.running = true;
  const interval = 1000 / state.settings.speed;
  state.intervalId = setInterval(() => tick(positions.backAttacks, state), interval);
  updateControlButtons();
}

function stopSimulation() {
  state.running = false;
  if (state.intervalId) {
    clearInterval(state.intervalId);
    state.intervalId = null;
  }
  updateControlButtons();
}

function stepSimulation() {
  if (state.terminated) return;
  if (state.running) stopSimulation();
  tick(positions.backAttacks, state);
}

function resetSimulation() {
  stopSimulation();
  init(positions.backAttacks);
  document.getElementById('event-log').innerHTML = '';
  appendLog(['[reset] All stages restored to controlled. Finish probability: 100%']);
}

function updateControlButtons() {
  const playBtn = document.getElementById('btn-play');
  playBtn.textContent = state.running ? '⏸' : '▶';
  document.getElementById('btn-reset').style.display = state.terminated ? 'inline-block' : 'none';
}
```

- [ ] **Step 2: Wire buttons to onclick handlers**

Add `onclick` attributes to the header buttons:
- Play/Pause button: `onclick="state.running ? stopSimulation() : startSimulation()"`
- Step button: `onclick="stepSimulation()"`
- Reset button: `onclick="resetSimulation()"`

- [ ] **Step 3: Verify in browser**

- Click ▶ — simulation runs, states animate, log fills
- Click ⏸ — simulation pauses
- Click ⏭ — advances one tick while paused
- Let simulation run until terminal (finish or bail) — verify Reset button appears
- Click Reset — all stages return to controlled, log clears, 100%

---

### Task 5: Settings Modal

Wire the gear icon to show/hide the settings modal, and connect sliders to the state.

**Files:**
- Modify: `BackAttacksViz.html`

- [ ] **Step 1: Wire modal show/hide**

```js
function toggleSettings() {
  const modal = document.getElementById('settings-modal');
  modal.style.display = modal.style.display === 'flex' ? 'none' : 'flex';
}
```

Gear icon: `onclick="toggleSettings()"`
Modal backdrop: `onclick="toggleSettings()"` (clicking outside closes)

- [ ] **Step 2: Wire slider inputs to state**

For each slider, add an `oninput` handler that:
1. Updates `state.settings[key]` with the slider value
2. Updates the displayed value label next to the slider
3. For speed slider: if simulation is running, clear and restart interval with new speed

```js
function updateSetting(key, value, displayEl) {
  if (key === 'speed') {
    state.settings.speed = parseFloat(value);
    displayEl.textContent = value + 'x';
    if (state.running) {
      clearInterval(state.intervalId);
      state.intervalId = setInterval(() => tick(positions.backAttacks, state), 1000 / state.settings.speed);
    }
  } else if (key === 'attackerSkill' || key === 'defenderSkill') {
    state.settings[key] = parseInt(value);
    displayEl.textContent = value;
  } else if (key === 'cascadeStrength') {
    state.settings.cascadeStrength = parseFloat(value);
    displayEl.textContent = value;
  } else if (key === 'bailThreshold') {
    state.settings.bailThreshold = parseFloat(value);
    displayEl.textContent = (parseFloat(value) * 100).toFixed(0) + '%';
  }
}
```

- [ ] **Step 3: Verify in browser**

- Click ⚙ — modal appears with 5 sliders at defaults
- Drag Speed to 5x — simulation visibly speeds up
- Drag Attacker Skill to 10 — stages tend to recover more
- Drag Defender Skill to 10 — stages tend to degrade more
- Drag Cascade Strength to 0.5 — cascading failures happen rapidly
- Drag Bail Threshold to 0.50 — bail triggers sooner
- Click outside modal — closes

---

### Task 6: Stage 5 Auxiliary + Terminal Animations

Add visual feedback for Stage 5 activation and finish/bail terminal states.

**Files:**
- Modify: `BackAttacksViz.html`

- [ ] **Step 1: Update render for Stage 5 and terminals**

In the `render()` function, add:
1. When `state.finishProb <= state.settings.bailThreshold * 1.5` (approaching threshold): Stage 5 footer border brightens gradually.
2. When terminal is bail: Stage 5 footer gets full brightness, tools highlight, a "BAIL" label appears.
3. When terminal is finish: all columns get a brief green flash, "FINISH" label appears in the finish meter area.

Use CSS classes toggled by render:
- `.auxiliary-warning` — border brightens to `#888`
- `.auxiliary-active` — border solid `#ccc`, tools brighten, label visible
- `.terminal-finish` — green pulse animation on finish meter
- `.terminal-bail` — red pulse animation on finish meter

- [ ] **Step 2: Add CSS for terminal animations**

```css
@keyframes pulse-green {
  0%, 100% { box-shadow: none; }
  50% { box-shadow: 0 0 20px rgba(106, 191, 106, 0.4); }
}
@keyframes pulse-red {
  0%, 100% { box-shadow: none; }
  50% { box-shadow: 0 0 20px rgba(191, 106, 106, 0.4); }
}
.terminal-finish { animation: pulse-green 1s ease-in-out 3; }
.terminal-bail { animation: pulse-red 1s ease-in-out 3; }
```

- [ ] **Step 3: Verify in browser**

- Set Defender Skill to 10, Cascade Strength to 0.5 — watch Stage 5 footer brighten as prob drops, then bail triggers with red pulse
- Reset, set Attacker Skill to 10 — watch finish trigger with green pulse
- Verify Reset clears all terminal visuals

---

### Task 7: Polish + Final Verification

Clean up edge cases, verify the complete experience.

**Files:**
- Modify: `BackAttacksViz.html`

- [ ] **Step 1: Event log styling**

Style log entries:
- Transitions: default color
- Finish prob changes: slightly dimmer
- Terminal events (★ FINISH, ✕ BAIL): bold, accent color
- Cascade annotations: italic
- Log container: max-height with overflow-y scroll, monospace font, dark background

- [ ] **Step 2: Finish meter polish**

- Bar fills from bottom, height proportional to finish prob (0-100%)
- Color gradient: green at top → red at bottom
- Dashed line at bail threshold position (updates when threshold slider changes)
- Percentage text updates with each render

- [ ] **Step 3: Verify complete flow end-to-end**

Open fresh page and verify:
1. Initial state: all controlled, 100%, no log entries
2. Click ▶ — simulation runs at 1x speed
3. States transition with smooth CSS animations
4. Tools highlight 1-2 at a time per transitioning stage
5. Finish meter bar rises and falls
6. Event log fills with formatted entries
7. Open settings ⚙, adjust sliders while running — changes take effect immediately
8. Eventually hits terminal (finish or bail)
9. Reset works cleanly
10. Step button works while paused

- [ ] **Step 4: Commit**

```bash
git add BackAttacksViz.html
git commit -m "feat: POC factored Markov visualization for BJJ Back Attacks"
```
