# 🧰 The Knife Rack (Prompt Library)

> "A craftsman is known by his tools, but a master is known by the way he chooses them."

This directory contains the core AI prompts that power the Latent Atelier workflow. They are designed to be highly constrained, deterministic, and biased towards engineering reality.

## 0. The Filter (Triage)

*Used for the initial "Gather -> Sort" phase.*

* [`triage-protocol.md`](./triage-protocol.md): The Dual-Stream evaluator (Strategic vs Tactical). Determines `mode/noise`, `mode/skim`, or `mode/deep`.
* [`reading-map.md`](./reading-map.md): The Navigator. Generates a non-linear reading path for high-signal papers.

## 1. The Lens (Decode)

*Zero-shot tools optimized for Raycast to instantly clear cognitive blockers.*

* [`raycast-translator.md`](./raycast-translator.md): High-precision ML translation (preserves engineering Chinglish).
* [`raycast-concept.md`](./raycast-concept.md): Unpacks jargon (Analogy -> Mechanism -> Definition -> Delta).

## 2. The Engine (Internalize)

*Deep-context tools optimized for Gemini Gems to deconstruct the paper.*

* [`gemini-math.md`](./gemini-math.md): The Architecture Mapper. Translates abstract LaTeX into physical system dependencies and readable logic.
* [`gemini-coder.md`](./gemini-coder.md): The Projector. Converts naive pseudocode into strictly shape-asserted PyTorch, exposing OOM risks.

---
*Note: Synthesis tools (The Socrates & Note Architect) are currently in local incubation and will be added in future versions.*
