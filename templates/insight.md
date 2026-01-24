---
title: "Insight: [The Specific Trade-off/Pattern Name]"
date: YYYY-MM
type: #Mechanism #Trade-off #Synthesis
tags: [Topic, Sub-problem]
status: Draft / Crystalized
---

<!-- markdownlint-disable MD013 -->

## 💡 Insight: [Title]

### 1. The Core Conflict (The Trigger)

> *Insight starts with a contradiction or a confusion.*

* **Observed Phenomenon:**
    [我看到了什么矛盾？例如：学术界都在推崇极其复杂的 RL 算法，但工业界（如 DeepSeek）似乎更偏向简单的规则强化。]
* **The Question:**
    [这一现象背后的核心疑问是什么？例如：Is algorithmic complexity actually inversely correlated with generalization in Reasoning tasks?]

---

### 2. Pattern Recognition (The Analysis)

*Select the paradigm that fits this insight.*

#### 🔍 Paradigm A: Mechanism (The "Same-Same" View)

* **Connecting the dots:**
    [Paper A 和 Paper B 看起来不同，但本质上是一样的。例如：They are both optimizing the lower bound of KL-divergence, just via different proxies.]
* **The Underlying Invariant:**
    [不变的那个数学/物理本质是什么？]

#### ⚖️ Paradigm B: Trade-off (The "Cost" View)

* **The Gain:** [使用该技术能得到什么？（如：精度提升）]
* **The Hidden Cost:** [必须付出的隐性代价是什么？（如：OOD 鲁棒性下降、推理延迟增加）]
* **The Boundary:** [在什么场景下，这个代价是不可接受的？]

#### 🔗 Paradigm C: Synthesis (The "Cross-Over" View)

* **Source Domain:** [灵感来源（如：搜广推）]
* **Target Domain:** [应用目标（如：LLM Alignment）]
* **The Transfer:** [我们能借用什么思想？]

---

### 3. The Judgment (The "So What")

> *This is the most important part. Your opinion.*

* **My Thesis:**
    [基于以上分析，我的判断是：[例如：For our specific stage, Data Curation is 10x more important than Algorithm Selection. We should stop fine-tuning PPO and focus on filtering the SFT dataset.]]
* **Engineering Principle:**
    [提炼成一条原则。例如：**"Better Data > Better Algorithm"** or **"Inference Compute > Training Compute"**.]

---

### 4. Evidence Support

* **Supporting Papers:** [Paper A], [Paper B]
* **Internal Experiments:** [我们的实验数据是否支持这个判断？]

<!-- markdownlint-enable MD013 -->
