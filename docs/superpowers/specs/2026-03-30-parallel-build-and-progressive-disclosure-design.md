# Parallel Build + Progressive Disclosure for Codebase-to-Course

**Date:** 2026-03-30
**Status:** Approved
**Problem:** Content-heavy codebases (e.g., openagentschool.org) cause generation failures and quality degradation. User feedback confirms the fix: "plan, split, parallelize and combine." The merged PR (#6) handled split and combine. This spec adds plan and parallelize, plus a restructure for token efficiency.

---

## 1. Goals

1. **Add a planning checkpoint** (Phase 2.5) that produces per-module briefs, enabling parallel writing
2. **Add parallel module writing** via subagents for complex codebases
3. **Restructure SKILL.md for progressive disclosure** — move heavy content into reference files so Claude (and subagents) only load what they need per phase
4. **Reduce default module count** from 5-8 to 4-6 to cut token cost and raise quality

## 2. Codebase complexity classification

The skill uses two paths based on what the codebase *is*, not file count:

| Type | Examples | Path |
|------|----------|------|
| **Simple** | Single-purpose CLI, small web app, one clear entry point, library | Sequential (no briefs, no agents) |
| **Complex** | Full-stack app, multiple services, content-heavy site, monorepo, 6+ modules needed | Parallel (briefs + subagents) |

Claude decides which path to take after Phase 2 (curriculum design), when it knows how many modules the course needs and how interconnected the codebase is.

## 3. New file structure

### Current
```
SKILL.md (303 lines — everything in one file)
references/
  design-system.md
  interactive-elements.md
  styles.css, main.js, _base.html, _footer.html, build.sh
```

### Proposed
```
SKILL.md (~200 lines — process skeleton + pointers)
references/
  content-philosophy.md    <- NEW: metaphors, quiz design, visual rules, tooltips
  gotchas.md               <- NEW: common failure points
  module-brief-template.md <- NEW: template for planning checkpoint
  design-system.md         (unchanged)
  interactive-elements.md  (unchanged)
  styles.css, main.js, _base.html, _footer.html, build.sh (unchanged)
```

### What loads when

| Phase | What Claude reads |
|-------|------------------|
| Phase 1 (Analysis) | SKILL.md only |
| Phase 2 (Curriculum) | SKILL.md only |
| Phase 2.5 (Briefs) — complex only | `module-brief-template.md` + `content-philosophy.md` |
| Phase 3 per module (sequential) | `content-philosophy.md` + `gotchas.md` + relevant sections of `interactive-elements.md` + `design-system.md` |
| Phase 3 per module (parallel agent) | Module brief + only the reference files tagged in the brief |

Parallel agents never read SKILL.md, the full codebase, or other modules' briefs.

## 4. Phase 2.5: Planning checkpoint (complex codebases only)

After curriculum design, Claude writes a module brief for each module. Briefs are saved to `course-name/briefs/` and can be deleted after assembly.

### Module brief contents

Each brief (~40-60 lines) contains:

- **Teaching arc** — metaphor, opening hook, key insight
- **Code snippets** — pre-extracted from the codebase, copy-paste ready with file paths and line numbers. This is the critical token-saving step: agents never re-read the codebase.
- **Interactive elements checklist** — which element types this module needs (chat, flow, quiz, translation, drag-and-drop) with enough detail to build them (actor names, question angles, flow steps)
- **Reference files to read** — which specific sections of `interactive-elements.md` and `design-system.md` the agent needs. Listed by section heading (e.g., "interactive-elements.md > Group Chat Animation, Quiz"). The agent reads only those sections, not the whole file.
- **Connections** — what the previous and next modules cover, so transitions are coherent across parallel agents

### Template

A `references/module-brief-template.md` file provides the structure. Claude fills it in per module during Phase 2.5.

## 5. Phase 3: Sequential vs. parallel build

### Sequential path (simple codebases)

Unchanged from current behavior. Claude writes modules one at a time in the main context. No briefs, no agents.

### Parallel path (complex codebases)

Modules are dispatched to subagents in batches of up to 3.

**Batching rules:**
- Max 3 modules per batch
- Short modules (3 screens, one quiz) can be paired — two briefs given to one agent
- Each agent receives only: its module brief(s) + the reference files listed in the brief

**What each agent does NOT get:**
- The full codebase (snippets are pre-extracted in the brief)
- SKILL.md (irrelevant — agent just writes HTML)
- Other modules' briefs
- Unneeded reference file sections

**After all agents finish:**
- Run `build.sh` to assemble `index.html`
- Main context does a consistency check: nav dots match modules, transitions make sense, no obvious tone shifts
- Open in browser for user review

## 6. Content moved out of SKILL.md

| Section | Destination | Lines saved |
|---------|-------------|-------------|
| Content Philosophy (metaphor rules, visual density rules, quiz design, tooltip guidance, code translation rules) | `references/content-philosophy.md` | ~58 |
| Gotchas (tooltip clipping, walls of text, recycled metaphors, scroll-snap, etc.) | `references/gotchas.md` | ~30 |

### What stays in SKILL.md
- Skill description and first-run welcome
- Who this is for + why this approach works
- The 4-phase process (now with sequential/parallel fork in Phase 3)
- Module structure table (as a menu, not a checklist)
- Mandatory interactive elements list
- Design Identity (short)
- Pointers to all reference files with descriptions of when to read them

### What changes in SKILL.md
- Default module count: 5-8 becomes **4-6**. "Most courses need 4-6 modules. Only go to 7-8 if the codebase genuinely has that many distinct concepts worth teaching. Fewer, better modules beat more, thinner ones."
- Phase 3 gains a decision point: "If this is a complex codebase (full-stack, multi-service, content-heavy, or 6+ modules), use the parallel path. Otherwise, stay sequential."

## 7. Token cost comparison

| Scenario | Before (estimated) | After (estimated) |
|----------|--------------------|--------------------|
| Simple 4-module course | Full SKILL.md (303 lines) + all references loaded 4x | Lean SKILL.md (~200 lines) + references loaded 4x |
| Complex 7-module course, sequential | Full SKILL.md (303 lines) + all references + full codebase in context for all 7 modules | Not applicable — complex courses use parallel path |
| Complex 7-module course, parallel | Not available | SKILL.md once for planning + 7 small briefs (~50 lines each) + targeted reference sections per agent |

The biggest savings come from parallel agents never loading the codebase or SKILL.md, and only loading the reference file sections they need.

## 8. Implementation plan (high-level)

1. Create `references/content-philosophy.md` — extract from SKILL.md
2. Create `references/gotchas.md` — extract from SKILL.md
3. Create `references/module-brief-template.md` — new file
4. Rewrite SKILL.md — slim down, add Phase 2.5, add sequential/parallel fork in Phase 3, update module count default, add reference file pointers
5. Test with a simple codebase (sequential path) — verify nothing broke
6. Test with a complex codebase (parallel path) — verify briefs + agents produce a coherent course
