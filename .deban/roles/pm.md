---
role: pm
owner: Gerald
status: active
last-updated: 2026-03-17
---

# Product Management

## Scope
Project goals, scope, priorities. What positions to map, what "interactive" means for the user, success criteria.

## Decisions
| Date | Decision | Rationale | Linked roles |
|---|---|---|---|
| 2026-03-17 | POC scope: Back Attacks first, Butterfly later (same structure, swap data) | Back Attacks is more clearly documented (Danaher source). Butterfly uses same model — just different data object. Ship one, validate, then duplicate. | [[dev]], [[arch]] |
| 2026-03-17 | "Interactive" = RNG simulation with play/pause/step + settings sliders + tool highlighting | Gerald clarified: user plays with sliders, watches RNG walk the parallel chains, sees tools light up representing the struggle. Not a drill/training mode. Not editable diagrams. | [[ux]], [[dev]] |
| 2026-03-17 | Structure is 4 control stages + 1 auxiliary (not 5 equal stages) | Stage 5 is structurally different — a bail-out decision point, not a Markov chain. The data model reflects this: `stages[]` array (4 items) + separate `auxiliary` object. | [[arch]] |

## Dead Ends
<!-- APPEND ONLY. Never delete. -->
| Date | What was tried | Why it failed / was rejected |
|---|---|---|

## Lessons
<!-- Distilled principles from Dead Ends. Written to be read cold. -->

## Open Questions
<!-- Format: - [ ] [question] — owner: [name] — since: YYYY-MM-DD -->
- [x] ~~The screenshots show numbered stages (1–5) but the brief says "not always in order"~~ — RESOLVED: parallel Markov chains model the non-linearity. Stages run simultaneously, not sequentially. — 2026-03-17
- [x] ~~What does "interactive" mean concretely?~~ — RESOLVED: RNG simulation with sliders, tool highlighting, play/pause. — 2026-03-17
- [ ] Is this a personal reference tool (Gerald only) or intended for sharing/teaching? — owner: Gerald — since: 2026-03-17
- [x] ~~The two examples both have 5 stages~~ — RESOLVED: 4 control stages + 1 auxiliary. Data model supports arbitrary stage count via `stages[]` array. — 2026-03-17
- [x] ~~Should state transitions/regressions be part of the data model?~~ — RESOLVED: yes, each stage is a 3-state Markov chain with explicit transition probabilities. — 2026-03-17

## Assumptions
- This is a POC. Architecture will likely change when integrated into a broader project.

## Dependencies
Blocked by:
Feeds into: [[arch]], [[dev]], [[ux]]

## Session Log
<!-- One line per session, newest first -->
2026-03-17 — Resolved 4/5 init questions. Scoped POC: Back Attacks first, sim + sliders, single HTML.
