---
title: Facts Writing Rule
description: Content and structure standards for the Facts system
---
# Facts Standard (Auto-Discovery Engine)

A **Fact** is a concise, highly practical technical insight aimed at a specific real-world problem. Thanks to our new 100% Dynamic Engine, you can easily create Tabs and manage their states while keeping your content perfectly clean and SEO-friendly.

## 1. Directory Structure
A Fact is not a single file. It is a **Directory** named after its URL slug.
Inside that directory, you can place as many `.md` files as you need. Every `.md` file automatically generates a new **Tab** in the interactive UI (except for `index.md`, which defines the main configuration).

`content/facts/[fact-slug]/`
- `index.md` (Required: Fact initialization & metadata)
- `solution.md` (Recommended)
- `why.md` (Optional)
- `how.md` (Any valid `.md` filename will automatically become a Tab)

---

## 2. File `index.md` (Brain Node)
This file is NEVER rendered as a Tab. It is strictly responsible for managing Metadata and displaying the primary Question Header at the top of the page.
It must start with a YAML block. 

```yaml
---
category: DevOps
tags: [nginx, sysadmin]
date: 2026-04-06
sections:
  - id: solution
    label: Solution
    icon: lightbulb
  - id: why
    label: Root Cause
    icon: help-circle
---
There is heavy incoming traffic. If I modify the config, is there a way to restart Nginx without dropping connections?
```
*(Notice the `sections` array: This array defines exactly which Tabs will be rendered and in what order. Each `id` must correspond to a `.md` file in the same directory, e.g., `solution.md` or `why.md`).*

---

## 3. Tab Files (`*.md`)
You are no longer restricted to a hardcoded list of valid filenames. You can create any `.md` file corresponding to the `id` you declared in the `sections` array of `index.md`.

Tab files are **100% PURE MARKDOWN**. They do not require any YAML frontmatter at the top because all metadata (`label`, `icon`, `order`) is already centrally managed inside your `index.md` file.

```markdown
The solution details are written directly here in Markdown without any YAML...
```

### Quick Shutdown
If you want to temporarily hide a Tab from the interface, you do NOT have to delete the `.md` file. Just completely remove its entry from the `sections` array in `index.md`. The UI system will cleanly ignore any Markdown file that is not explicitly registered in `index.md`.

---

## 4. Section Philosophy

There is **no fixed template** for which sections a fact must have. Every fact is unique and the content dictates which sections make sense. Any section type can appear in any fact — for example, a Playground tab is not exclusive to frontend facts; a Python algorithm, a SQL query optimizer, or a Kubernetes config can all have interactive playgrounds if it helps the learner.

### Minimum Requirement
Every fact MUST have at least:
- `solution.md` — The primary answer (every question deserves at least one answer)
- `source.md` — Official documentation references. Always use `order: 99` so it appears last

### Section Catalog
Below is a **catalog of possible section types**. Pick whichever sections make sense for your fact. You are NOT limited to this list — invent new section types freely. Each `.md` file = one Tab.

| Section Type | Suggested File | Icon | When to Use |
|-------------|---------------|------|-------------|
| Solution / Answer | `solution.md` | `Lightbulb` | The primary answer. Every fact needs this |
| Why / Root Cause | `why.md` | `HelpCircle` | When understanding the "why" is as important as the fix |
| How / Step-by-step | `how.md` | `Wrench` | When the solution needs a detailed walkthrough |
| Implementation | `implementation.md` | `Code2` | Production-ready code with explanation |
| Alternative Approaches | `alternatives.md` | `GitBranch` | When multiple valid solutions exist (common in interviews) |
| Common Pitfalls | `pitfalls.md` | `AlertTriangle` | Anti-patterns, mistakes people make |
| Example | `example.md` | `Box` | Concrete example, demo, sample project |
| Playground / Interactive | `playground.md` | `Play` | Any fact can have this: live code sandbox, algorithm visualizer, SQL query tester, config simulator, etc. |
| Debugging | `debugging.md` | `Bug` | How to diagnose in production, DevTools walkthrough |
| War Stories | `war-stories.md` | `Flame` | Real production incidents, postmortem lessons |
| Checklist | `checklist.md` | `ListChecks` | Before/after checklist, decision matrix |
| Comparison | `comparison.md` | `Scale` | Trade-off tables, "X vs Y" analysis |
| Advanced | `advanced.md` | `Rocket` | Deep dive beyond the basic answer |
| Patterns | `patterns.md` | `Layers` | Related design patterns, combined strategies |
| Libraries & Tools | `libraries.md` | `Package` | Production-ready libraries, framework integrations |
| Config / YAML | `config.md` | `Settings` | Configuration files, infrastructure setup |
| Cost Analysis | `cost-math.md` | `Calculator` | Real cost calculations, before/after numbers |
| Prevention | `prevention.md` | `ShieldCheck` | How to prevent the problem from happening again |
| Source / References | `source.md` | `BookOpen` | Official docs, RFCs, specifications. Always `order: 99` |

### Naming Your Own Sections
You can name sections anything — the filename becomes the internal identifier, the `label` frontmatter becomes the tab title:

```
react-useeffect-infinite-loop/
  index.md
  solution.md          → label: "Solution"
  advanced-fixes.md    → label: "Advanced Fixes"
  prevention.md        → label: "Prevention"
  source.md            → label: "Source"

kubernetes-hpa-autoscaling-pitfalls/
  index.md
  solution.md          → label: "Solution"
  custom-metrics.md    → label: "Custom Metrics"
  advanced.md          → label: "KEDA & Predictive"
  source.md            → label: "Source"
```

### Guiding Principles
- **Let the content decide the structure.** Don't force a "why" tab if the question is self-explanatory. Don't skip a "playground" tab if a live demo would be 10x more useful than text.
- **Any section type can appear in any fact.** A playground is not "a frontend thing" — a Python sorting algorithm, a PostgreSQL query plan, or a Docker networking demo can all benefit from an interactive tab. A war-stories tab is not "a DevOps thing" — a CSS layout bug that took down production is a valid war story too.
- **More tabs = more depth.** The tabbed UI is designed for quick switching. Prefer 5 focused tabs over 2 giant tabs.
- **Each tab = one focused concept.** If a tab covers two different things, split it into two tabs.
- **Use `order` gaps** (1, 3, 5, 10, 99) to leave room for future insertions without renumbering everything.
- **Interview-tagged facts benefit from multiple solution tabs.** Candidates who present 4-5 approaches score higher than those with 1 answer. Each approach CAN be its own tab (e.g., `dataloader.md`, `caching.md`, `redis-lock.md`).

---

## 5. Writing Style Context
- **No Fluff**: Skip extensive greetings. Jump straight into the answer. The core principle of a "Fact" is extreme conciseness.
- **Bullet Points**: Prefer making short, structured lists over large text paragraphs.
- **Clear Code**: All code snippets must be wrapped in proper Code Blocks specifying the language (e.g., ````nginx````). Syntax highlighting is built directly into the engine.

---

## 6. Batch Generation Workflow

When generating facts from `plan.md` or writing a new fact, follow this process:

1. **Create the directory** at `content/facts/[slug]/`
2. **Write `index.md`** with proper frontmatter (category, tags, date, and `sections` array) and the primary question text.
3. **Write `solution.md`** — the primary answer (pure Markdown).
4. **Decide which additional sections** fit this specific topic — consult the Section Catalog but don't limit yourself to it.
5. **Write `source.md`** with official documentation links.
6. **Final Step: Run the Engine** — You MUST execute `node scripts/generate-facts.js` in the primary workspace. This script will read your new `index.md` and update the master Central Manifesto (`content/facts/index.md`) for the whole project.

> Minimum viable fact = `index.md` + `solution.md` + `source.md` (3 files).
> A good fact typically has 4-7 tabs, but there is no upper limit.
