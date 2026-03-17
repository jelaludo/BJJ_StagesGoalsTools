---
role: ux
owner: Gerald
status: active
last-updated: 2026-03-17
---

# UX Design

## Scope
Visual layout and interaction patterns for the interactive diagrams. How stages, goals, and tools are presented and navigated.

## Decisions
| Date | Decision | Rationale | Linked roles |
|---|---|---|---|
| 2026-03-17 | Vertical columns layout: each stage is a column, states stack top-to-bottom, tools below, finish meter on right | Alternatives rejected: horizontal rows (closer to original screenshots but harder to see parallel nature), hybrid columns + SVG graph (too complex for POC). Vertical columns read naturally as parallel independent chains. | [[dev]], [[arch]] |
| 2026-03-17 | Monochrome dark theme + semantic state accents only | Gerald rejected per-column color coding as "too distracting." Base theme: dark (#0c0c0c), Georgia serif, monochrome greys. Color only on the active state: green (controlled #6abf6a), grey (contested #bbb), red (lost #bf6a6a). Inactive states dim (#444). | [[dev]] |
| 2026-03-17 | Tool highlighting: active tool = brighter text + visible border, inactive = dim | During simulation, 1-2 tools per stage light up on transition, representing the technique being used. Tools stay highlighted until next transition in that stage. | [[dev]] |
| 2026-03-17 | Settings via gear icon → modal overlay, not inline panel | Keeps the main view clean. 5 sliders update in real time. | [[dev]] |

## Dead Ends
<!-- APPEND ONLY. Never delete. -->
| Date | What was tried | Why it failed / was rejected |
|---|---|---|
| 2026-03-17 | Per-column color coding (blue for grips, orange for hips, purple for head, red for strangulation) | Gerald found it too distracting — pulled focus from structure and animation to decoration. Replaced with monochrome + state-only accents. |

## Lessons
<!-- Distilled principles from Dead Ends. Written to be read cold. -->

## Open Questions
<!-- Format: - [ ] [question] — owner: [name] — since: YYYY-MM-DD -->

## Assumptions

## Dependencies
Blocked by: [[arch]], [[pm]]
Feeds into: [[dev]]

## Session Log
<!-- One line per session, newest first -->
2026-03-17 — Layout decided (vertical columns), visual style decided (monochrome + state accents). Per-column colors rejected.
