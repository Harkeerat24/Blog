---
title: "Structured Format for Saved Circuit Data | GSoC 2026 | Phase 1 Report"
date: 2026-07-12T18:12:22Z
author: Harkeerat Singh
image: "/images/Harkeerat_Singh/cover_image.png"
tags: ["GSoC 2026", "CircuitVerse", "Vue.js", "TypeScript", "JSON", "Serialization", "Algorithms", "Graph Theory"]
draft: false
type: post
---

Hello everyone, I'm [Harkeerat Singh](https://www.linkedin.com/in/harkeerat-singh-cse/) ([Harkeerat24](https://github.com/Harkeerat24)) and this is my mid-term blog post for Google Summer of Code 2026 with [CircuitVerse](https://circuitverse.org) for the project **Structured Format for Saved Circuit Data**.

---

## Project Overview

Here's the problem the project set out to fix: two logically identical circuits, saved in a different order, had been producing completely different save files. Because there was no canonical representation, identical circuits did not hash the same, could not be reliably diffed, and could not be trusted for features that depend on stable output.

The goal of this project is to design and implement a canonical JSON format that produces **identical output for any two logically equivalent circuits**, in the Vue-based simulator ([cv-frontend-vue](https://github.com/CircuitVerse/cv-frontend-vue)).

---

## The Community Bonding Period

The Community Bonding period kicked off on **May 2nd, 2026** with a meeting where GSoC contributors, mentors, and maintainers met. The atmosphere was incredibly warm, relaxed, and welcoming. It was more like a friendly hangout than a formal call.

On **May 9th, 2026**, I had my project kickoff meeting with my mentors ([Aboobacker](https://github.com/tachyons), [JoshVarga](https://github.com/JoshVarga), [Arnabdaz](https://github.com/Arnabdaz), and [Aryann](https://github.com/aryanndwi123)). We went through the project timeline, discussed our serialization algorithm design in depth, and aligned on how the canonical format should handle complex edge cases. It was a fun, highly insightful session that left me with a clear direction for the weeks ahead.

During this period, I worked on closing a few issues in [cv-frontend-vue](https://github.com/CircuitVerse/cv-frontend-vue) that may come in handy later in the project. Here are the merged PRs from that work:

- **[PR #1046](https://github.com/CircuitVerse/cv-frontend-vue/pull/1046)** — **Merged** — Fixed the insert subcircuit dialog to prevent a subcircuit from being inserted into itself (or a circular chain), which was causing a cyclic dependency.
- **[PR #1081](https://github.com/CircuitVerse/cv-frontend-vue/pull/1081)** — **Merged** — Made the input field auto-focus as soon as a modal dialog opens, so users can start typing right away without an extra click.
- **[PR #1084](https://github.com/CircuitVerse/cv-frontend-vue/pull/1084)** — **Merged** — Replaced loose equality (`==`) with strict equality (`===`) in the circuit selection logic to fix incorrect matches caused by type coercion.

---

## Phase 1 Sprint Log

### Sprint 1 (**25–31 May, 2026**): Implementing the Export Pipeline

The first week was dedicated to implementing the core export pipeline and canonicalization logic in `canonical.ts`.

The core of the canonicalization pipeline utilizes the **1-Dimensional Weisfeiler-Leman (1-WL)** graph algorithm to generate a structural fingerprint for each component. This ensures that component sorting is independent of the sequence in which components were originally placed or saved.

- **[PR #1093: gsoc26: Implementing canonical pipeline for deterministic json](https://github.com/CircuitVerse/cv-frontend-vue/pull/1093)**.

### Sprint 2 (**1–7 June, 2026**): Scaling to Multi-Circuit Projects

Once the single-circuit case was stable, the focus shifted to extending the canonical format to support **multi-circuit projects**. While basic multi-circuit serialization came together smoothly, the more challenging aspect, which is subcircuit dependency ordering, was deferred to the following week.

- **[PR #1094: gsoc26: multi circuit support in canonical.ts](https://github.com/CircuitVerse/cv-frontend-vue/pull/1094)**.

### Sprint 3 (**8–14 June, 2026**): Subcircuits and Topological Sorting

This week was dedicated to solving the **subcircuit dependency** challenge: if Circuit B uses Circuit A as a subcircuit, Circuit A must be processed before Circuit B to guarantee determinism.

Two key mechanisms made this possible:

- **Kahn's Algorithm** for topological sorting over the scope dependency graph: We compute the in-degrees of each circuit and process zero-dependency scopes first. By using a sorted queue, tie-breaks between simultaneously-ready scopes are resolved deterministically.
- **Cycle Detection**: as a natural side effect of Kahn's algorithm, if there's a circular dependency (e.g., Circuit A and Circuit B depend on each other), the queue can't fully drain. This is detected and thrown as an error.

![Canonical JSON output](/images/Harkeerat_Singh/week3_canonical_json_demo.png)

- **[PR #1095: feat: added subcircuit support in canonical.ts using khans algorithm](https://github.com/CircuitVerse/cv-frontend-vue/pull/1095)**.

> By the end of this week, `canonical.ts` was capable of generating deterministic canonical JSON for any valid layout — single circuit, multi-circuit, or nested subcircuits.

### Sprint 4 (**15–21 June, 2026**): Building the Import Path

This sprint was slower than planned — I was unwell for a good part of it. Even so, I managed to draft most of `importCanonical.ts`, the file responsible for reconstructing the circuit on the canvas from the canonical JSON generated by `canonical.ts`. The planned JSON Schema validation work got pushed to the next sprint.

### Sprint 5 (**22–28 June, 2026**): Import Pipeline Completed

During Sprint 5, I completed the implementation of `importCanonical.ts`. The import pipeline handles:

- **Full multi-circuit support** on import.
- **Deterministic subcircuit reconstruction**, using the same Kahn's algorithm implementation to ensure nested subcircuits are initialized before their parent circuits.
- **Round-trip verification**: re-exporting the newly imported scope to confirm its hash matches the original.
- **Deferred JSON Schema validation**: held back until the canonicalization and import pipelines are reviewed by the mentors and the format itself is settled.

![Week 5 Round-Trip Verification Check](/images/Harkeerat_Singh/week5_roundTripCheck.png)

- **[PR #1131: feat: add canonical JSON import pipeline with round-trip verification](https://github.com/CircuitVerse/cv-frontend-vue/pull/1131)**.

### Sprint 6 (**29 June – 12 July, 2026**): First Merge & End-to-End Export/Import

My mentors went through the existing PRs I'd raised, caught a couple of issues, and floated a great optimization idea along the way: **caching canonical hashes** so unmodified circuits can skip the full pipeline on save. That's now planned for the second half. I worked through their feedback, resolved the open PR conversations, and got **[PR #1095](https://github.com/CircuitVerse/cv-frontend-vue/pull/1095)** merged!

With that foundation in place, my focus shifted to hooking the engine up to the real [cv-frontend-vue](https://github.com/CircuitVerse/cv-frontend-vue) frontend.

**On the import side**, I updated the workflow to accept the new deterministic canonical structure and dropped support for the legacy layout parsing. I also added a backup — if a canvas import crashes or fails for any reason, the system automatically rolls back and preserves the user's existing work instead of wiping the canvas.

**On the export side**, I tied the "Export as File" action directly to `canonical.ts` so it outputs clean `.cv` files. To make the feature more developer-friendly, I added:

- A live JSON preview panel
- A Copy to Clipboard button
- A Download Circuit button

![Export JSON dialog with live CodeMirror preview](/images/Harkeerat_Singh/week7_export_import_ui.png)

- **[PR #1132: feat: update Export Project dialog and import project for canonical support](https://github.com/CircuitVerse/cv-frontend-vue/pull/1132)**

---

## Demo Video

This video showcases all the progress made during the first phase of the project:

{{< youtube hPdCzJOewT8 >}}

---

## Current Status

| PR | Title | Status |
|----|-------|--------|
| [#1093](https://github.com/CircuitVerse/cv-frontend-vue/pull/1093) | gsoc26: Implementing canonical pipeline for deterministic json | Closed — superseded by [#1095](https://github.com/CircuitVerse/cv-frontend-vue/pull/1095) |
| [#1094](https://github.com/CircuitVerse/cv-frontend-vue/pull/1094) | gsoc26: multi circuit support in canonical.ts | Closed — superseded by [#1095](https://github.com/CircuitVerse/cv-frontend-vue/pull/1095) |
| [#1095](https://github.com/CircuitVerse/cv-frontend-vue/pull/1095) | feat: added subcircuit support in canonical.ts using khans algorithm | **Merged** |
| [#1131](https://github.com/CircuitVerse/cv-frontend-vue/pull/1131) | feat: add canonical JSON import pipeline with round-trip verification | Open — in review |
| [#1132](https://github.com/CircuitVerse/cv-frontend-vue/pull/1132) | feat: update Export Project dialog and import project for canonical support | Open — in review |
| — | JSON Schema and validation | In Progress |
| — | Writing tests | In Progress |

> **Note**: The current pull requests are raised in the `v1` of the repository. This lets us test the canonicalization and import pipelines thoroughly without any risk of breaking the main simulator workflow.

---

## Lessons & Learning

- **TypeScript from day one.** Writing the file in `.ts` directly from the beginning was a great decision. The compile-time type safety it introduced made refactoring complex serialization logic much safer and is overall much better for the project's long-term maintenance.
- **Mentor reviews clear doubts and open new doors.** Active discussions and review feedback did not just help identify edge cases; they opened up new directions and resolved conceptual doubts before they became bugs.
- **The right algorithm solves two problems at once.** Choosing Kahn's algorithm for subcircuits resolved loading dependencies and gave us cycle detection out of the box.
- **Loose types hide design decisions.** Replacing `unknown` with strict typings forced us to define the format's actual shape, catching bugs at compile-time.
- **Stacked PRs keep momentum high.** Splitting development into chained git branches meant we could keep coding without waiting on reviews.

---

## Next Steps

- **JSON Schema Validation**: Write `canonicalSchema.json` and integrate validation into `importCanonical.ts`.
- **Hash Caching**: Implement canonical hash caching to bypass redundant computations on subsequent saves.
- **Testing**: Write comprehensive unit tests for the pipelines.
- **Auto-Layout with ELK.js**: Integrate the `ELK.js` library to automatically generate readable layouts for circuits imported without positioning format.
- **Idempotent Converter**: Build a migration tool to convert legacy `.cv` files into the new format.
- **Canonical-to-Verilog Exporter**: Implement a single-pass Verilog code generator.

---

## Acknowledgements

I'm incredibly grateful to my mentors [Aboobacker](https://github.com/tachyons), [JoshVarga](https://github.com/JoshVarga), [Arnabdaz](https://github.com/Arnabdaz), and [Aryann](https://github.com/aryanndwi123) for their guidance, feedback, and support throughout the first half of the project. I also want to thank the CircuitVerse community for their warm welcome and collaborative environment.

I'm looking forward to a productive and exciting second half of GSoC!
