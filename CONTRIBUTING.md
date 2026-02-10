# 🕯️ The Studio Protocol

> "We shape our tools and thereafter our tools shape us."

This document outlines the **rituals** and **mechanisms** behind Latent-Atelier. It explains how we transform raw information noise into crystallized intelligence.

---

## 1. The Materials (Issue Taxonomy)

In this atelier, GitHub Issues are not "bugs"—they are **Raw Materials**. We use labels to sort the incoming stream before it hits the workbench.

### 🟢 Raw Clay (Capture)

*Tag what you found. Objective & Immediate.*

* **`type: paper`**: Academic signals. New arXiv papers or tech reports.
* **`type: tool`**: The instruments. Libraries (vLLM), frameworks, or codebases.
* **`type: insight`**: Sparks. Blog posts, tweets, or fleeting thoughts worth capturing.
* **`type: log`**: The Observatory. Monthly compilations & reports.

### 🔵 The Kiln (Processing)

*Decide the Mode of processing. Subjective & Deliberate.*

* **`mode: deep`**: **(The Chisel)**. High-signal items. Requires a dedicated branch, atomic notes, and a PR.
* **`mode: quick`**: **(The Sketch)**. Good for trend tracking. Findings are summarized directly into the `Monthly Intelligence Log`.
* **`mode: noise`**: **(The Scrap)**. Low signal. Evaluated and explicitly discarded.

*(Note: We do not use a `wip` label. To mark an item as active, **Assign** it to yourself.)*

### 🟠 Housekeeping (Meta)

*Keeping the studio clean and directed.*

* **`meta: plan`**: **(The Compass)**. Quarterly roadmaps, research focus, or strategic direction.
* **`meta: bug`**: Typos, broken links, or rendering glitches.
* **`meta: chore`**: Structure adjustments, template tuning, or "cleaning the floor".

---

## 2. The Rituals (Workflow)

Our process mimics the craftsman's loop: **Gather → Sort → Sculpt → Display**.

### Phase I: Foraging (Mobile/Web)

* **The Act**: Creating an Issue is purely for **Capture**.
* **The Rule**: Drop the link, tag the `type`. Do not overthink. Treat it as tossing a stone into the basket.

### Phase II: Curation (Weekly Review)

* **The Act**: Review the "Inbox". Distinguish signal from noise and assign a **Mode**.
* **The Decision**:
  * Is it foundational? → Label **`mode: deep`**.
  * Is it news? → Label **`mode: quick`**.
  * Is it noise? → Label **`mode: noise`** + Comment "Why" + **Close as Not Planned**.

### Phase III: Sculpting (Execution)

* **Activation**: When you start working, **Assign** the Issue to yourself.
* **Execution**:
  * **Deep Mode**: Checkout a topic branch (e.g., `topic/deepseek-analysis`). Write the "Note" file. Submit a PR that links to the Issue.
  * **Quick Mode**: Extract the insight into the `Monthly Intelligence Log`. Comment on the Issue.
* **Synthesis**: All valid knowledge eventually merges into `main`.

### Phase IV: The Gallery (Archive)

* **Closure**:
  * **Deep**: Automatically closed when the PR is merged.
  * **Quick**: Manually closed after content is migrated to the Log.
* **Permanence**: We link to **local files** or **original source URLs** in our reports, never to the temporary Issues. The artifact must stand alone.

---

## 3. The Architecture (Branching)

* **`main` (The Gallery)**:
  The public showroom. History here is clean, linear, and polished. Only finished works (merged PRs) enter here.
* **`topic/*` (The Workbench)**:
  Where the mess happens. `topic/paper-reading`, `topic/concept-proof`. Atomic commits are encouraged here.
* **`fix/*` (The Broom)**:
  Quick fixes for typos or structural errors.

---

<small><i>"Simplicity is the ultimate sophistication."</i></small>
