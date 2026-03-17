---
role: arch
owner: Gerald
status: active
last-updated: 2026-03-17
---

# Architecture

## Scope
Data model for the Stages/Goals/Tools structure. Tech stack selection. How diagrams are stored, rendered, and extended to new positions.

## Decisions
| Date | Decision | Rationale | Linked roles |
|---|---|---|---|
| 2026-03-17 | Factored Markov model: 4 parallel 3-state mini-chains (controlled ⇄ contested ⇄ lost) per position | A flat Markov chain (like the GameLoops project) would explode to 243 composite states for 5 stages × 3 states. Factored model keeps each dimension tractable while capturing the parallel nature of BJJ control. Alternatives rejected: flat chain (combinatorial explosion), simple checklist (doesn't capture the dynamic loss/recovery of control). | [[dev]], [[ux]] |
| 2026-03-17 | State-based cascading dependencies between stages | Losing upstream stages continuously pressures downstream stages each tick (not just on transition). Back Attacks cascade: Grip→Hip,Head; Hip→Head,Strangle; Head→Strangle. Alternative rejected: independent chains (misses the reality that losing grips unravels everything). Also rejected: event-based cascade (one-tick bump too weak to feel realistic). | [[dev]] |
| 2026-03-17 | Single monolithic HTML file, no framework | Matches existing MarkovChainViz.html pattern. POC — may change architecture when integrating into broader project. Alternatives rejected: modular HTML+JSON (unnecessary separation for POC, needs local server), framework-based React/Svelte (overkill). | [[dev]], [[devops]] |
| 2026-03-17 | Finish probability = product of per-stage weights (controlled=1.0, contested=0.4, lost=0.05) | Multiplicative model means losing even one stage has dramatic effect, which matches BJJ reality. Stage 5 auxiliary triggers when composite drops below configurable threshold (default 0.25). | [[dev]] |

## Dead Ends
<!-- APPEND ONLY. Never delete. -->
| Date | What was tried | Why it failed / was rejected |
|---|---|---|

## Lessons
<!-- Distilled principles from Dead Ends. Written to be read cold. -->

## Open Questions
<!-- Format: - [ ] [question] — owner: [name] — since: YYYY-MM-DD -->
- [ ] Transition probability tuning — base values are educated guesses, will need playtesting — owner: Gerald — since: 2026-03-17

## Assumptions
- The 3-state model (controlled/contested/lost) is sufficient granularity per stage. No evidence this needs more states, but untested.
- Multiplicative finish probability produces realistic-feeling dynamics. Unverified until POC runs.

## Dependencies
Blocked by:
Feeds into: [[dev]], [[ux]]

## Session Log
<!-- One line per session, newest first -->
2026-03-17 — Designed factored Markov model, cascade mechanics, data model. Spec written and reviewed.
