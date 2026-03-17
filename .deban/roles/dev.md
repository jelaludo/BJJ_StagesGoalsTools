---
role: dev
owner: Gerald
status: active
last-updated: 2026-03-17
---

# Development

## Scope
Implementation of interactive diagrams from static BJJ position maps. Front-end rendering, data model, interactivity.

## Decisions
| Date | Decision | Rationale | Linked roles |
|---|---|---|---|
| 2026-03-17 | Single HTML file, all CSS/JS inline, position data as JS objects at top | Matches MarkovChainViz.html pattern from GameLoops project. No build step, no dependencies, double-click to open. Easy to duplicate for Butterfly by swapping data object. | [[arch]] |
| 2026-03-17 | Simulation engine: setInterval tick loop, single-roll-three-outcomes per stage per tick | Each tick: calculate cascade modifiers → roll transitions → highlight tools → recalculate finish prob → check terminals → log. Contested state resolves via single random number against three mutually exclusive outcomes (recover/degrade/stay). | [[arch]] |
| 2026-03-17 | Settings modal with 5 real-time sliders: speed, attacker skill, defender skill, cascade strength, bail threshold | Skill sliders use ±0.03 per point above/below 5, clamped [0.05, 0.95]. Allows tuning feel without editing code. | [[ux]] |

## Dead Ends
<!-- APPEND ONLY. Never delete. -->
| Date | What was tried | Why it failed / was rejected |
|---|---|---|

## Lessons
<!-- Distilled principles from Dead Ends. Written to be read cold. -->

## Open Questions
<!-- Format: - [ ] [question] — owner: [name] — since: YYYY-MM-DD -->
- [ ] Tool selection logic — currently random 1-2 tools per transition. Could be weighted or contextual later. — owner: Gerald — since: 2026-03-17
- [ ] Finish condition — "95% for 3 consecutive ticks" is arbitrary, may need tuning — owner: Gerald — since: 2026-03-17

## Assumptions

## Dependencies
Blocked by: [[arch]]
Feeds into: [[qa]]

## Session Log
<!-- One line per session, newest first -->
2026-03-17 — Tech decisions made: single HTML, tick loop engine, settings modal. Spec written.
