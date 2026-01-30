---
title: "Anthropic Agent Skills"
authors: [Anthropic]
date: 2026-01-28
layer: AGENT
tags: [Agent]
url: "https://github.com/anthropics/skills"
---

## 📄 Anthropic Agent Skills

### 1. TL;DR (Executive Summary)

![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs20260129224115943.png)

* **The Paper's Pitch:**
 Anthropic 提出了 Agent Skills，针对复杂 Agent 任务下的上下文问题（Context Bloat），通过渐进式披露（Progressive Disclosure）机制，将工具定义上下文消耗大幅降低。
 它是*程序性知识 (Procedural Knowledge)*的标准化载体，允许 Agent 在不消耗大量上下文的前提下，拥有“无限”的技能库。
* **Our Takeaway:**
  为业务 Agent 的 Token 成本与延迟提供了新的分层架构方案：核心在于将“能力 MCP”与“程序性知识 SKills”解耦，MCP 负责标准化的 I/O 连接，而 Skills 负责将 SOP 转化为按需加载的认知显存。
  * 何时用 MCP: 需要连接数据、原子操作、且逻辑简单时。
  * 何时用 Skills: 需要遵循 SOP、多步推理编排、或长尾低频任务时。一个高质量 Skills 必须包含*清晰的触发条件 (YAML)*、*结构化的思维链 (Instruction)* 以及*自我修正机制(Validation)*。

---

### 2. Background & Context

* **Status Quo:** 目前主流的 Agent 构建方式是 Eager Loading，即在 Prompt 中一次性注入所有可用工具的 Schema 定义。MCP 虽然解决了工具连接的标准化问题，但它默认的交互方式仍然依赖于将工具列表全量推送给模型。
* **The Gap:** 随着工具数量增加，上下文窗口面临复杂度爆炸带来的“噪声”问题。注入过多的工具定义不仅导致“Attention Dilution”，降低了模型在任务上的推理准确率，同时带来 Time-To-First-Token（TTFT）延迟和推理成本。之前 MCP 本身关注的是互操作性，并未在协议层解决上下文管理的问题。

### 3. Motivation & Intuition

* **Author's Hypothesis:**
  Anthropic 工程团队的核心假设是：Agent 的能力上限不应受限于 Context Window 的长度，而应取决于 **有效注意力（Effective Attention）** 的分配。
  * 痛点：认知负荷过载。传统的 Eager Loading（全量加载）会导致 Prompt 充满无关的工具定义，产生噪音干扰模型决策。
  * 本质：这本质上是一个信息检索 (IR) 与状态管理问题。
  * 解决：引入 **“按需认知 (Just-in-Time Cognition)”**。通过文件系统遍历与分层检索，Agent 仅在任务触发特定意图时，才动态挂载（Mount）相关的“操作手册”。这实现了认知资源的“冷热分离”。
* **Our Resonance:**
  在实际业务中，常面临“有工具、无策略”的困境：
  * 能力-意图断层 (Capability-Intent Gap)：
    * 可能我们有 Naive / Graph / Agentic RAG 等技术上能力互补的 MCP 工具。但面对具体问题时，模型并不清楚何时该“互补使用”，往往随机选择。
  * 决策成本与不确定性：
    * 为了教模型怎么用，我们过去被迫在 System Prompt 中写入冗长的 ⁠If-Else 逻辑。这不仅增加了 Token 成本，还因缺乏“渐进式披露”机制，导致模型在多工具场景下的决策变得不可预测。
  * Skills的介入：
    * Skills 充当了 **“策略层”。它不改变底层工具代码，而是通过注入一段“专家逻辑”**（例如：“先用图谱找实体关系，若无结果，再降级为向量检索”），以低成本的方式接管了模型的控制流。

| 维度 | Custom Instructions | Projects (Knowledge) | MCP | Agent Skills (Procedures) |
| :--- | :--- | :--- | :--- | :--- |
| 工程定位 | Global Config | Static Context | Hardware Abstraction | User Space Logic |
| 功能 | 风格与全局约束 | 静态背景资料库 | 外部世界的手与眼 (I/O) | 完成任务的大脑皮层 (SOP) |
| 加载模式 | Always-on | Pre-loaded | Schema 常驻 | 动态激活 |
| 典型载体 | System Prompt | PDF/Docs | API Servers | `SKILL.md` / Scripts |

### 4. Methodology (The "How")

* **Core Architecture:**
  ![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs20260129231051069.png)
  * Layer 1 (Discovery): `SKILL.md` 的 Frontmatter (YAML) 充当索引。模型启动时，只读取极简的 `name` 和 `description` (~100 tokens)。
  * Layer 2 (Activation): 当模型判断需要使用某个工具时，通过文件读取工具加载 `SKILL.md` 的 Markdown 正文。
  * Layer 3 (Execution): 执行脚本或调用底层的 MCP 工具。
* **Key Algorithms:**
  * Progressive Disclosure:
    $$Cost_{total} = C_{base} + \sum_{i \in Active} C_{skill\_i}$$
    对比传统模式的 $Cost_{total} = C_{base} + \sum_{all} C_{tool}$，该算法将复杂度从工具总数解耦，仅与当前活跃任务相关。 ![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs20260129230916221.png)
  * Orchestration via Filesystem: 利用模型对代码库和文件系统的天然理解力，将“工具调用”转化为“文件阅读与脚本执行”。
* **Crucial Details:**
  ![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs20260129231243275.png)
  ![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs20260129231408347.png)
  ![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs20260129231550212.png)
  ![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs20260129231808502.png)

### 5. Experiments & Verification

* **Settings:**
  * 基座模型：Claude 系列模型，重点观察不同规模模型在处理技能 metadata 时的语义匹配精度
  * 评测基准：
    * 任务闭环率：在跨越 PDF 处理、Excel 自动化及代码重构的多步骤任务中，评估其最终产出的准确性。
    * 指令对齐度：区分“高自由度（启发式建议）”与“低自由度（硬性脚本约束）”环境下的执行偏差。
    * Token 负载效率：评估初始系统提示词的压缩比。
  * 环境设置：
    * 文件系统驱动的虚拟机：实验环境提供完整的虚拟机访问权限。
    * Bash 交互机制：Claude 并不在内部“拥有”技能，而是通过标准的 Unix 工具（如 cat、grep 或 ls）从文件系统中读取操作手册。
    * 运行时限制： 明确区分环境边界。在 Claude API 环境下，遵循无网络访问、无运行时包安装的强约束，仅依赖预配置的依赖库（如 pdfplumber、openpyxl）。
* **Main Results:**

| 评估维度 | Prompt-only | Agent Skills | 架构洞察 |
| :--- | :--- | :--- | :--- |
| 初始 Token 消耗 | 随功能线性增长（通常 >10,000 tokens） | 极低固定负载（仅 ~100 tokens 元数据） | 实现了 95% 以上的初始负载削减 |
| 推理一致性 | 易受无关指令干扰，产生“长上下文幻觉” | 强隔离性（仅在触发时加载相关逻辑） | 模块化隔离显著提升了任务稳定性 |
| 技能扩展性 | 受限于上下文窗口（难以超过 20 个复杂任务） | 理论上无限扩展（可挂载 100+ 技能） | 突破了上下文窗口的物理限制 |
| 环境依赖处理 | 需在 Prompt 中详细描述 API 调用逻辑 | 确定性脚本执行（直接调用预设 Python/Bash） | 将概率性生成转变为确定性执行 |

* **Ablation Study:**
  * Progressive Disclosure —— 真正的“英雄模块 (Hero)”： 这是 Agent Skills 的架构基石。它通过三级加载机制（Level 1：元数据发现；Level 2：指令加载；Level 3：资源/脚本执行）彻底释放了上下文窗口。它是实现“100+ 技能挂载”而无性能衰减的唯一原因。 这种机制确保模型仅在“意识到”需要某项能力时，才通过 Bash 读取相关的 SKILL.md。
  * Deterministic Code Execution:在处理高风险任务（如数据库迁移、复杂数学计算）时，直接调用预设的工具脚本比让模型“即兴”生成代码的错误率降低了几个数量级。它提供了模型无法逾越的“逻辑围栏”。
  * Feedback Loops: 包含“Run-Validate-Fix”流程的技能，其任务成功率远高于纯文本指令。这种“闭环验证”是减少幻觉、确保产出符合生产规范的最后一道防线。

### 6. Conclusion & Limitations

* **Paper's Conclusion:** Agent Skills 是对 MCP 的补充而非替代。MCP 提供标准化的连接 (Connection)，Skills 提供结构化的流程 (Workflow)。二者结合是构建 Real-World Agent 的有效路径。
* **Our Verification / Plan:**
  * Limitation:
    * Latency: 每次激活 Skill 都需要额外的 `read_file` 轮次，增加了交互延迟。
    * Governance: 基于文件的 Skills 分散在各处，难以像 API 网关一样集中治理和鉴权。
    * Skill Amnesia: 随着对话进行，加载的 Skill 内容可能被挤出上下文，需要 Context Pinning 机制。

---

### 7. Insights & Spark (The "Soul")

* **Deep Reflection:**
  这标志着 Agent 开发从 `Prompt Engineering` 正式迈向 `Context Engineering`。Skills 的本质是将 Procedural Memory 从模型参数或 Prompt 中剥离，固话为一种中间态的代码文档。意味着不再强求模型“记住”复杂的业务逻辑流程，而是教模型“查阅”流程手册。这与人类专家解决陌生问题的路径（查文档->执行）是完全同构的。
  ![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs20260129231954581.png)

* **Business Connection:**
  * SOP-as-Code 2.0: 从“文本约束”到“可执行资产”
    * 误区澄清：传统的“SOP 即 Prompt”是将长文本一次性塞入系统，Token 消耗大且逻辑由于上下文稀释而容易失效。
    * 新范式：Skills 将 SOP 变成了代码级的可交付物。
      * 原子化：独立于模型存在，是一个独立文件
      * 可测试：可以像测试代码一样，针对 Skill 写单元测试
  * Enterprise SPM (Skill Package Manager): 业务逻辑的供应链管理
    * 定义：参考`npm`或`pip`，团队内部建立`SPM`，用于管理“认知模块”的依赖与分发
  * KaaS (Knowledge as a Service): 不仅卖工具，更卖“方法论”
    * 定义：SaaS 卖的是“锤子”（工具/MCP），KaaS 买的是“老木匠的手艺”（Skill）

### Appendix

#### Awesome Agent Skills

[awesome-claude-skills](https://github.com/ComposioHQ/awesome-claude-skills)
[skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)

#### Idea

1. spec coding
2. report generation
3. app(automation)
