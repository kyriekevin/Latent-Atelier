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

### 🔵 The Kiln (Triage)

*Decide how to process it. Subjective & Deliberate.*

* **`queue: deep dive`**: **(The Chisel)**. High-signal items requiring deep study, a dedicated branch, and atomic notes.
* **`queue: quick`**: **(The Sketch)**. Good for trend tracking. Destined for a brief mention in the Monthly Observatory.
* **`queue: wip`**: Currently on the workbench.

### 🟠 Housekeeping (Meta)

*Keeping the studio clean.*

* **`meta: bug`**: Typos, broken links, or rendering glitches.
* **`meta: chore`**: Structure adjustments, template tuning, or "cleaning the floor".

---

## 2. The Rituals (Workflow)

Our process mimics the craftsman's loop: **Gather → Sort → Sculpt → Display**.

### Phase I: Foraging (Mobile/Web)

* **The Act**: Creating an Issue is purely for **Capture**.
* **The Rule**: Drop the link, tag the `type`. Do not overthink. Treat it as tossing a stone into the basket.

### Phase II: Curation (Weekly Review)

* **The Act**: Review the "Inbox". Distinguish signal from noise.
* **The Decision**:
  * Is it foundational? → Mark as `queue: deep-dive`.
  * Is it news? → Mark as `queue: quick`.
  * Is it noise? → **Close** (Discard).

### Phase III: Sculpting (Execution)

* **Deep Work**: Checkout a topic branch (e.g., `topic/deepseek-analysis`). Write the "Note" file. This is where *The Clay* becomes *The Chisel*.
* **Synthesis**: Merge findings into the `Monthly Intelligence Log`.

### Phase IV: The Gallery (Archive)

* **Closure**: Once the knowledge is crystallized into a Markdown file, the Issue (Ticket) is **Closed**.
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
