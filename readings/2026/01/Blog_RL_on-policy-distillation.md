---
title: "On-Policy Distillation"
authors: [Thinking Machines Lab]
date: 2026-01-25
layer: ALIGN
tags: [On-Policy, Distillation]
url: "https://thinkingmachines.ai/blog/on-policy-distillation/"
---

## 📄 On-Policy Distillation

### 1. TL;DR (Executive Summary)

![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs/20260130155727809.png)

* **The Paper's Pitch:**
    针对传统 SFT 因 Off-Policy 训练导致的 Exposure Bias 和 Distribution Shift，以及 RL 因稀疏反馈导致的 Sparse Reward 和高方差训练难题，提出了针对 LLM Post-Training 阶段的 On-Policy Distillation 范式。
    该范式通过强制 Student 模型在自身生成轨迹上采样，并由 Teacher 模型提供稠密的 Token-level Reward（Logits 或 Discriminator Score），融合了“RL的分布鲁棒性”与“SFT的高样本效率”。
    实验结果表明，在推理密集型任务上（数学推理、代码生成）和指令遵循任务上，能以仅 RL 方法 1/10 的计算成本达到甚至超越 RL 的效果，同时在持续学习场景下有效克服了灾难性遗忘问题。
* **Our Takeaway:**
    该思路标志着 Post-Training 技术栈从 Blind Imitation via Off-Policy SFT 向 Introspective Learning via On-Policy Distillation 演进。
    **核心价值**: 对于构建高性能垂类小模型而言，该路径通过“Token 级稠密反馈”解决了长程推理中的信用分配难题。这是复刻如 o1 或 DeepSeek-R1 等推理模型“Aha Moment”的核心，使小模型能够精准捕获思维链中关键的逻辑转向，而非仅仅死记硬背标准答案。
    **指导意义**：这不仅仅是蒸馏技术的升级，本质上是一种 Dense Reward RL（稠密奖励强化学习）。它预示着未来的小模型训练将不再依赖静态数据集，而是依赖“动态生成的自我轨迹 + 强模型的实时纠偏”，即 Compute is the new Data。

---

### 2. Background & Context

LLMs 能够在特定领域表现出专家级性能，往往需要一系列训练方法支持，大致可以分为三个主要阶段：

* **Pre-Training**: 学习通用能力，如语言使用、广泛推理和世界知识。
* **Mid-Training**: 学习领域知识，例如代码、医学数据库或公司内部文档。
* **Post-Training**: 引导出目标行为，例如遵循指令、解决数学问题或进行聊天。

业务上往往需要构建垂类小模型，会在开源模型基础上进行 Post-Training 来达到特定的业务目标。

![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs/20260130155837307.png)

当前 LLM Post-Training 技术栈中存在的结构性矛盾——一个被称为“不可能三角”的困境：在样本效率 （Sample Efficiency）、分布一致性 （Distributional Consistency）和信号密度（Signal Density）之间，无法同时取得优异表现。

* **Status Quo:** 目前业界的主流 Post-Training 分为两种范式：
  * **Off-Policy**：给定输入prompts，让学生模型去学习模仿来自其他来源（例如教师模型）的生成序列（rollout）。
  * **On-Policy**：给定输入prompts，从学生模型自身采样来生成序列（rollout），并为这些序列分配奖励（reward）。

    以下面的数学题为例 `prompt: What is 5 + (2 x 3)?`

  * 常用方式一：
    ![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs/20260130155913357.png)
    * 可以使用 RL 进行 On-Policy Training
    * 机制：On-Policy。模型必须根据当前策略 $\pi_\theta$ 采样生成完整回复 y，然后由环境 （Reward Model）给予反馈。
    * 数学本质：最大化 Expected Reward $\mathcal{J}(\theta) = \mathbb{E}_{x \sim \mathcal{D}, y \sim \pi_\theta(y|x)} [R(x, y)]$。
    * 优势：解决了分布偏移问题，因为训练和推理时模型面对的状态分布是一致的。

  * 常用方式二：
    ![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs/20260130160003331.png)
    * 通常使用 SFT 完成 Off-Policy 训练
    * 机制：Off-Policy。模型在训练过程中看到的数据完全来自于 Ground Truth （老师的历史轨迹），而非自己生成的。
    * 数学本质：最小化 Forward KL Divergence (前向 KL 散度) $\mathcal{L}_{\text{SFT}} = D_{\text{KL}}(\pi_{\text{ref}} \parallel \pi_\theta) = \mathbb{E}_{x \sim \mathcal{D}, y \sim \pi_{\text{ref}}(y|x)} [-\log \pi_\theta(y|x)]$。
    * 优势：样本效率高，训练收敛速度快。

* **The Gap:** 尽管 SFT 和 RL 代表的 Off-Policy 和 On-Policy 构成了当前主流范式，但是实际应用面临无法忽视的痛点，这也是 On-Policy Distillation 试图填补的空白。
  * SFT: Exposure Bias 和 Compounding Errors
    * 现象：在 SFT 训练时，模型每预测一个 token，输入的上文（Context）都是完美的“正确答案”（Teacher Forcing）。然而在推理（Inference）阶段，模型必须基于自己生成的上文进行预测。一旦模型在第 $t$ 步犯了一个小错，第 $t+1$ 步的输入就变成了模型从未见过的“错误状态”（Out-of-Distribution）。由于模型从未在训练中学习过如何从错误中恢复，这个微小的偏差会被迅速放大，导致后续生成的整个序列逻辑崩塌。
    * 本质：SFT 就像是老师手把手教写字，老师写一笔，学生描一笔。学生从未体验过“写错一笔后该如何救回来”的过程。因此，SFT 模型在长程推理（Long-context Reasoning）任务中往往表现脆弱。
    * *Notes*: 在这种学习方式下观察到的另一个问题是：学生模型可能只是学会模仿教师的风格而不一定能学会真正的能力提升（《The False Promise of Imitating Proprietary LLMs》）。
  * RL: Sparse Reward 和 Credit Assignment
    * 现象：虽然 RL 是 On-Policy 的（学生自己写作业），但反馈信号极其稀疏。通常模型生成了 500 个 token 的长文，Reward Model 只会在最后给出一个标量分数（Scalar Reward，例如 +1 或 -1）。
    * 本质：这是一个典型的 Credit Assignment Problem。模型不知道这 500 个 token 中，究竟是哪一个 token 导致了低分。是逻辑错了？还是语法错了？为了从这种稀疏信号中学习，RL 算法需要海量的采样（Sample Complexity 极高）来通过蒙特卡洛方法估计梯度，导致训练成本通常是 SFT 的数倍，且对超参数极其敏感，训练极不稳定。
    * *Notes*: 这种反馈的稀疏性导致 RL 在许多应用中效率低下。
  * SeqKD:
    * 传统的序列级知识蒸馏（Sequence-Level Knowledge Distillation, SeqKD）虽然利用了 Teacher 的 Logits，但大多仍然是在固定的离线数据集上进行的（Off-Policy）。这意味着它依然无法解决 Student 和 Teacher 之间的能力差距导致的分布不匹配问题（Train-Test Mismatch）。当 Student 的能力远弱于 Teacher 时，仅仅在 Teacher 擅长的区域进行拟合，并不能帮助 Student 在自己生成的“充满未知”的轨迹中找到方向。

  * The Opportunity: 能否结合 On-Policy 的探索能力（解决分布偏移和暴露偏差）与 Teacher Logits 的稠密信号（解决稀疏奖励和样本效率问题）？这正是 On-Policy Distillation 诞生的技术原点。

### 3. Motivation & Intuition

模型只有在修正自己生成的错误时，才能真正掌握知识。

* **Author's Hypothesis:** Experience with Guidance
    问题的本质在于“静态数据”与“动态能力”之间的错位。
  * Teacher as a GPS, not a Driver
    在 SFT 中，Teacher 是司机，Student 只是坐在副驾驶观察的乘客。无论观察多少次，乘客都无法学会处理突发的路况。
    在 On-Policy Distillation 中，Student 被推到了驾驶座上（On-Policy Sampling），开始自己驾驶。而 Teacher 坐在副驾驶，手里拿着高精地图（Logits），实时、逐秒地提供反馈：“向左一点”、“油门太大了”。
    * 直觉：这种实时反馈（Token-level Feedback）的信息量（Bits per episode）是 $O(N)$，其中 $N$ 是序列长度；而 RL 的反馈信息量仅为 $O(1)$。这就解释了为什么 On-Policy Distillation 能极大提升样本效率。
  * The "Reverse KL" Intuition: Mode-Seeking vs. Mean-Seeking
    这是一个非常关键的数学直觉，决定了模型的行为特征。
    * **SFT (Forward KL $D_{KL}(P || Q)$)**: 倾向于 Zero-Avoiding。为了最小化 Forward KL，只要 Teacher 分布 $P$ 的概率不为零，Student 分布 $Q$ 的概率也必须不为零。如果 Teacher 的分布具有多样性（例如，对一个问题有 A、B、C 三种正确的解法），SFT 训练出来的 Student 往往会尝试“平均”这三种解法，结果生成出一种语法通顺但逻辑混乱的混合物（Mean-Seeking）。
    * **On-Policy Distillation (Reverse KL $D_{KL}(Q || P)$)**: 倾向于 Mode-Seeking。为了最小化 Reverse KL，Student 分布 $Q$ 只能在 Teacher 分布 $P$ 概率不为零的区域有值。这迫使 Student 放弃“平均主义”，而是必须锁定（Lock-in）Teacher 认为概率最高的那一个解法模式。对于推理任务（Math, Code）而言，这种“锐化”（Sharpening）特性至关重要——宁愿模型掌握一种解法并做到极致，也不愿它在三种解法中犹豫不决。

* **Our Resonance:**
  * "The SFT Ceiling": 在微调私有数据时，SFT 很容易过拟合。一旦数据量不够，模型就会变成“鹦鹉学舌”，遇到稍微变体的 Prompt 就无法泛化。
  * "Catastrophic Forgetting": 在垂直领域 SFT 后，模型往往会遗忘通用能力。

On-Policy Distillation 提供了一种 "Online Synthetic Data" (在线合成数据) 的全新视角：最好的训练数据不是存储在硬盘里的静态文件，而是模型在当前参数状态下，为了纠正自身错误而动态生成的那些轨迹。这与 Data-Centric AI 的理念殊途同归——Data is not static; it is a function of the model's current policy.

### 4. Methodology (The "How")

* **Core Architecture:**
    1. Rollout (Student Generation):
        学生模型 $\pi_{\theta}$ 接收 Prompt，基于当前的参数 $\theta$，自回归地采样生成完整的轨迹序列 $\hat{y} = (y_1, y_2, \ldots, y_T)$。
        * Crucial Detail: 这里通常使用非零的 Temperature (e.g., $T=1.0$) 来保证探索性（Exploration），防止模型陷入局部最优。
    2. Evaluation (Teacher Guidance):
        教师模型 $\pi_{teacher}$ (或 Discriminator) 观察学生生成的完整轨迹 $\hat{y}$。注意，Teacher 在这里不需要生成文本，只需要对 Student 生成的文本进行“批改”：
    3. Feedback (Dense Reward Signal):
        * White-Box Setting (Standard/GKD): 教师计算在 $\hat{y}$ 上每一步的条件概率分布（Logits）$\pi_{teacher}(y_t | x, y_{<t})$。
        * Black-Box Setting (GAD): 当无法获取 Logits 时，训练一个判别器 $D$，它给出标量打分 $R$。
    4. Update (Policy Gradient Optimization):
        计算梯度并更新 $\theta$，使 Student 的未来生成更接近 Teacher 的偏好。

  * Case Study
    ![image.png](https://raw.githubusercontent.com/kyriekevin/img-hosting/main/imgs/20260130160042925.png)
    * 颜色越深代表该处token受到教师模型惩罚越高
    * 可以发现，该方法惩罚了那些导致学生产生偏离的短语起始token，直观上对应于引导推理的重要“分岔token”
    * 可以发现：最终答案虽然是错的但并未受到惩罚，因为在给定整个前述序列的条件下，它是可预测的

    > LLM 的本质并非平滑的序列生成，而是由极少数高熵功能点（Function Tokens）支撑的非线性决策
    > 1. 分岔归因： 推理的成败取决于短语起始的“决策 Token”。训练应惩罚逻辑偏离的起始点，而非末端的错误答案。末端错误往往是路径偏移后的必然结果（高概率、低熵），惩罚它是无效的；只有纠正分岔点，才能改变推理轨迹。
    > 2. 能力锚点： 模型的推理 Feature 并非弥散在全文，而是高度压缩在Function Tokens（如格式标识、逻辑转折）中。
    > 3. 80/20 效率： 80% 的 Token（内容词/结果词）仅是逻辑惯性的填充，不承载能力；只有 20% 的高熵决策 Token 才是模型智能的真正载体。
    > 训练的本质是在正确的分岔点（高熵 Token）激发出正确的逻辑标识（Function Token），而非对全文本的统计拟合。

```python
# Initialize teacher client (main):
teacher_client = service_client.create_sampling_client(
    base_model=teacher_config.base_model,
    model_path=teacher_config.load_checkpoint_path,
)

# Sample trajectories (main):
trajectories = do_group_rollout(student_client, env_group_builder)
sampled_logprobs = trajectories.loss_fn_inputs["logprobs"]

# Compute reward (compute_teacher_reverse_kl):
teacher_logprobs = teacher_client.compute_logprobs(trajectories)
reverse_kl = sampled_logprobs - teacher_logprobs
trajectories["advantages"] = -reverse_kl

# Train with RL (train_step):
training_client.forward_backward(trajectories, loss_fn="importance_sampling")
```

* **Key Algorithms:**
  * Reverse KL (White-Box)
    核心目标是最小化 Student 分布相对于 Teacher 分布的 **Reverse KL 散度**。与传统的监督微调（Forward KL）不同，它让 Student 在自己生成的分布上进行学习。
    * 数学目标 (Mathematical Objective)
        $$ \mathcal{L}_{RKL}(\theta) = \mathbb{E}_{x \sim \mathcal{D}} [ D_{KL}(\pi_\theta(\cdot|x) \parallel \pi_{teacher}(\cdot|x)) ] $$
        展开到 Token 级别：
        $$ \mathcal{L}_{RKL}(\theta) = \mathbb{E}_{x \sim \mathcal{D}} \mathbb{E}_{y \sim \pi_\theta} \left[ \sum_t \left( \log \pi_\theta(y_t | x, y_{<t}) - \log \pi_{teacher}(y_t | x, y_{<t}) \right) \right] $$

    * 强化学习视角 (RL Mapping)
        通过 Policy Gradient (REINFORCE) 的标准推导，最小化上述 Loss 的梯度等价于：
        $$\nabla_\theta \mathcal{L}_{RKL}(\theta) = \mathbb{E}_{y \sim \pi_\theta} \left[ \nabla_\theta \log \pi_\theta(y) \cdot \text{Advantage}(y) \right]$$

        在这种映射下，我们可以拆解出两个关键概念：
      * Token-level Reward ($r_t$): $\log \pi_{teacher}(y_t) - \log \pi_\theta(y_t) - 1$
        * 这是一个即时反馈，衡量 Student 采样的 Token 是否符合 Teacher 的分布。
      * Analytical Advantage ($A_t$): $\log \pi_{teacher}(y_t) - \log \pi_\theta(y_t)$
        * 定义：优势函数衡量一个动作比平均水平“好”多少。在这里，它直接由两个分布的 Logits 差值决定。

    * 深度直觉 (Deep Insight)
        通过这个 **Token-level Advantage**，我们可以清晰地看到训练过程中的行为：
      * Case A (Positive Reinforcement):
        * 条件：$\pi_{teacher} > \pi_\theta$（Teacher 认为这个 Token 概率很高，但 Student 还没意识到）。
        * 结果：$A_t > 0$。梯度更新会**增加**该 Token 的生成概率。这不仅是模仿，更是根据 Teacher 的知识进行“纠偏”。
      * Case B (Negative Punishment):
        * 条件：$\pi_\theta > \pi_{teacher}$（Student 产生了盲目自信的 Hallucination，或者生成了 Teacher 认为低质量的词）。
        * 结果：$A_t < 0$。梯度更新会**严厉惩罚**该 Token 路径。由于 RKL 的特性，这种惩罚极其敏感，能迅速剪掉不符合 Teacher 分布的生成分支（Mode-seeking 行为）。

    * 效率提升 10x-100x？
      * 解析优势 (Analytical vs. Sampled):
        * 传统 RL 的 Advantage 是靠采样和 Value Network “猜”出来的，方差极大。
        * RKL 的 Advantage 是**直接计算**出来的（White-box Logits），信号极其精准，方差极低。
      * 稠密信号 (Dense Signal):
        * 传统 RL 通常是序列末尾给一个 Reward（稀疏信号），模型需要艰难地进行“信用分配”（Credit Assignment）。
        * RKL 在**每个 Token** 位置都有独立的 Reward。模型每走一步都知道这一步走得好不好，不存在延迟反馈问题。
      * 全分布监督:
        * 传统 RL 只有 1 bit 的反馈（对或错）。
        * RKL 提供了整个词表的概率分布对比，Student 一次迭代获得的知识量远超传统 RL。

  * Generative Adversarial Distillation (GAD, black-box setting)
    在现实场景中，最强的 Teacher 往往是闭源 API 模型（如 GPT-4o, Claude 3.5 Sonnet），无法获取 Logits（White-box 信息）。GAD 借鉴 GAN 的思想，将蒸馏转化为一个对抗博弈问题。

    * The Minimax Game
        GAD 引入一个参数化的判别器 $D_\phi$（Discriminator），目标是区分文本是由 Teacher 生成的还是 Student 生成的。

    * Discriminator Step (判别器更新):
        $$ \max_\phi \mathbb{E}_{y \sim \pi_{teacher}} [\log D_\phi(x, y)] + \mathbb{E}_{y \sim \pi_\theta} [\log(1 - D_\phi(x, y))] $$
      * 目标：最小化分类误差。使 $D_\phi$ 成为一个极其敏锐的“考官”，能够从细微的语气、逻辑差异中识别出 Student 的作品。
    * Student Step (生成器更新):
        $$ \min_\theta \mathbb{E}_{y \sim \pi_\theta} [\log(1 - D_\phi(x, y))] \quad \text{或更常用的} \quad \max_\theta \mathbb{E}_{y \sim \pi_\theta} [\log D_\phi(x, y)] $$
      * 目标：提高生成质量，使生成的文本被判别器误认为是 Teacher 创作的。

    * Insight: Why it works?
      * 自适应的 Reward Model (The Moving Target):
        在黑盒场景下，我们没有 $D_{KL}$ 可算。但判别器 $D_\phi$ 的输出分数实质上充当了 **On-Policy Reward**。
        * 如果 $D_\phi(x, y)$ 接近 1，意味着判别器认为这是 Teacher 写的 $\to$ 给 Student 高 Reward。
        * 如果 $D_\phi(x, y)$ 接近 0，意味着判别器识破了 Student $\to$ 给 Student 负反馈。
      * 天然的课程学习 (Curriculum Learning):
        随着 Student 的能力增强，判别器也必须变得更强才能将其区分开。这种“水涨船高”的关系保证了反馈信号始终处于 **"Goldilocks Zone"**（不过于简单，也不至于难到无法收敛），避免了静态 Reward Model 容易被 Student 刷分（Reward Hacking）的问题。
      * 解决 Distribution Drift:
        传统的黑盒蒸馏（如简单的模仿学习）只学习 Teacher 的结果，而不处理 Student 偏离分布后的表现。GAD 通过 **On-Policy 采样**，让判别器在 Student 自己的分布上进行打分，迫使 Student 在偏离正确路径时受到惩罚，从而解决分布偏移问题。
    * GAD 可以理解为一种 “以 Teacher 为基准的自适应 RLHF”
      * 它解决了 Rule-based Reward 无法覆盖主观任务的问题（比 GRPO 更开放）
      * 它解决了静态 Reward Model 容易被过度优化（Reward Hacking）的问题
      * 虽然信号不如 White-box 稠密（Token-level 较弱），但它是目前白嫖闭源模型（黑盒）最高效的对齐手段

  * GOLD (Cross-Tokenizer Adaptation)
    当 Teacher (如 Qwen) 和 Student (如 Llama) 属于不同家族时，其词表（Vocabulary）完全不兼容。直接计算 $D_{KL}$ 在数学上是未定义的，因为相同 ID 对应的 Token 不同（如 ID 128 在 A 中是 "cat"，在 B 中可能是 "dog"）。

  * 维度与语义的不对称
    * 维度不一致：$|V_{S}| \neq |V_{T}|$（词表大小不同）。
    * 语义偏移：即使维度相同，同一词汇在两个词表中的索引（Index）也完全不同。

  * Cross-Vocabulary Mapping
    * Step 1: Vocabulary Alignment (Set Intersection)
        首先通过字符串匹配，找到两个词表在文本空间上的交集 $V_{inter} = V_S \cap V_T$。只有在这个集合中的 Token，其概率分布才具有直接对齐的基础。
    * Step 2: Probabilistic Mapping (概率重分布)
        对于在 $V_{inter}$ 中的 Token，我们将 Teacher 的概率分布进行重新缩放（Renormalization），使其在 Student 的词表空间内也有对应的响应。
        $$ P_{aligned} = \text{Softmax}(\text{Logits}_{Teacher} \text{ mapped to } V_{inter}) $$
    * Step 3: Soft Alignment & ULD (退避策略)
        对于无法直接通过字符串对应的 Token，GOLD 通常采用 **ULD (Universal Logit Distillation)** 或 **Token Embedding 相似度**来进行平滑：
      * Embedding Mapping: 如果词表不匹配，通过计算 Student Embedding 空间与 Teacher Embedding 的余弦相似度，将 Teacher 的 Logits 投影到 Student 空间。
      * Soft Selection: 选取 Student 中语义最接近的 Top-K 个 Token 来分摊 Teacher 的概率质量。

* **Crucial Details:**
    1. Temperature Management & Softmax Scaling
        * **探索 vs. 学习:**
            在采样阶段，Student 通常使用 $T = 1$ 以保证生成的**探索性**。
        * **分布平滑 (Dark Knowledge):**
            Teacher 的原始 Logits 分布往往极其尖锐（Peaky），直接计算 KL 散度可能导致梯度爆炸。
        * **Trick:** 在计算 KL 时，对 Teacher Logits 应用较高的 Temperature（如 $T=2.0$）进行 **Softmax Temperature Scaling**。
            * **原理：** 平滑后的分布能显现出那些概率较小但非零的 Token 之间的相对关系（即 **Dark Knowledge**）。
            * **效果：** 提供了比 Hard Label 更丰富的监督信号，使梯度更新更稳健。

    2. Memory & Efficiency Optimization (vLLM Integration)
        * **On-Policy 蒸馏的最大瓶颈在于 Generation Latency（生成延迟）**。
        * **Actor-Learner 架构:**
            在 PyTorch 训练循环中直接调用 `.generate()` ，由于 Python GIL 和显存切换开销，效率极低。
            * **Best Practice** 是采用解耦架构：
                * **Actor (Inference Engine):** 使用 vLLM 或 TGI 等高性能引擎进行异步 Rollout，生成数据推入 Replay Buffer。
                * **Learner (Training Engine):** 进程从分布式训练 Buffer 中读取数据，进行梯度更新。
        * **KV Cache Management:**
            **必须复用生成阶段的 KV Cache**。如果计算 Loss 时重新对序列进行 Forward，计算量将翻倍。在 DeepSpeed 或 TRL 等框架中，通过精细管理 Cache 传递提升效率。

    3. Data Mixture (GKD's $\lambda$ Parameter)
        纯 On-Policy 训练面临 **Cold Start Problem**：训练初期 Student 生成的文本可能是逻辑混乱的乱码，Teacher 此时的反馈（Advantage）质量极差。

        * **Solution: 混合策略 (GKD 方案)**
            通过参数 $\lambda$ 控制 **On-Policy** 数据与 **Off-Policy (Ground Truth)** 数据的比例。
            * **$\lambda$ 的意义：** 决定了训练序列中有多少比例来自于 Teacher 的直接引导（SFT 数据），多少来自于 Student 的自我探索。
            * **典型配置：** 初期设置较大的 $\lambda$（如 0.5），随着训练进行逐渐减小 $\lambda$（**Curriculum Learning**）。
        * **收益：** 这种“先扶着走，再放手跑”的策略，既利用了 Ground Truth 保证训练不坍塌，又利用了 On-Policy 解决了分布偏移（Distribution Drift）问题。

### 5. Experiments & Verification

* **Settings:**
  * Tasks (Benchmarks):
    * Mathematical Reasoning: GSM8K, MATH (考察多步推理和逻辑链)。
    * Instruction Following: AlpacaEval, MT-Bench (考察对话质量和多样性)。
    * Summarization: XSum (考察信息提取和抗幻觉能力)。
    * Coding: HumanEval, MBPP。
  * Models:
    * Teachers: GPT-4 (Black-box), Qwen-2.5-72B, GPT-5-Chat (GAD paper reference).
    * Students: Llama-3-8B, Qwen-2.5-14B, TinyLlama-1.1B, PaLM-2-S。
  * Baselines:
    * SFT (SeqKD): 传统的离线蒸馏。
    * RLHF (PPO): 标准的 Proximal Policy Optimization。
    * Best-of-N (Rejection Sampling): 推理时采样 N 个选最好的。

* **Main Results:**

| Metric / Method | SFT (SeqKD) | RLHF (PPO) | On-Policy Distillation (GKD/GAD) |
| --- | --- | --- | --- |
| Reasoning Score (e.g., MATH) | Baseline (e.g., 35%) | +2~5% (e.g., 38%) | +4~8% (e.g., 42%) |
| Sample Efficiency (Data) | High (1x) | Very Low (100x samples) | Medium-High (5-10x samples) |
| Compute Cost (Training) | Low (Only Backprop) | Very High (RM + PPO) | Medium (Rollout dominant) |
| Convergence Speed (Steps) | Fast | Very Slow | Fast (7-10x faster than RL) |
| Output Diversity | Low (Mode collapse) | High (Exploration) | Sharpened (Mode-seeking) |
| OOD Robustness | Poor (Exposure Bias) | Good | Excellent |

* Result 1: Efficiency Breakthrough (Thinking Machines): 在数学推理任务（AIME scores）上，On-Policy Distillation 达到与 RL 相当甚至更好的分数，所需的 Gradient Steps 少了 7-10倍。累计算力消耗（考虑到 Teacher Inference Cost）总体减少 50-100倍。这证明了 Dense Reward 的巨大价值。
* Result 2: Black-Box Power (GAD): 在 LMSYS-Chat 盲测中，Qwen-2.5-14B (Student) 使用 GAD 蒸馏后，与其 Teacher (GPT-5-Chat) 的胜率持平。这极其惊人，因为它意味着仅仅通过判别器的二分类信号，学生就能完全模仿老师的“神韵”（Nuances），而不需要具体的 Logits 数值。这打破了“必须有 White-box Access 才能做好蒸馏”的迷信。
* Result 3: Continual Learning Resilience: Thinking Machines 的实验展示，当模型在特定内部文档（Internal Docs）上微调时，SFT 模型的通用指令遵循能力（IFEval）大幅下降（灾难性遗忘）。而采用 On-Policy Distillation（同时以原始通用模型作为 Teacher），Student 几乎完全保持了通用能力，同时掌握了新知识。这是因为 Teacher（作为锚点）始终在约束 Student 的行为边界，防止其在参数空间中漂移过远。

* **Ablation Study:**

  * On-Policy vs. Off-Policy Data: 实验一致表明，只用 Off-Policy 数据（即标准 SeqKD），效果会有明显上限。一旦引入 On-Policy 数据（哪怕只有 20%），性能曲线就会突破 SFT 的平台期。On-Policy 数据是突破上限的关键。
  * Reverse KL vs. Forward KL: 在生成任务上，Reverse KL 显著优于 Forward KL。Forward KL 容易导致 Student 生成“幻觉”（Hallucinations），因为它不严厉惩罚 Student 在 Teacher 概率低的地方生成内容；而 Reverse KL 对此有极大的惩罚（Zero-forcing property），从而有效抑制了幻觉。

### 6. Conclusion & Limitations

Thinking Machines 团队特别强调其在 Personalization（个性化） 和 Continual Learning（持续学习） 场景下的统治力——它允许模型像人类一样，通过不断的“试错-纠正”循环来适应新环境，而不仅仅是背诵新数据。

* **Paper's Conclusion:**
    1. Compute Bound by Inference (推理计算瓶颈): 训练吞吐量（Tokens/sec）受限于 Student 的推理速度（Generation）。如果 Student 生成很慢，或者 Teacher 的 Logits 计算很慢，整个训练管线会被 Inference 阻塞。这比 SFT 这种纯 Forward/Backward 过程要慢得多，对工程架构提出了极高要求。
    2. Teacher Dependency (教师依赖): 模型的上限由 Teacher 决定。如果 Teacher 本身有严重的 Bias 或逻辑漏洞，On-Policy Distillation 会极其高效地将这些缺陷“刻入”Student 的骨髓（Reverse KL 的 Mode-Seeking 特性会加剧这一点）。它无法像 RL 那样通过 Ground Truth Reward（如代码执行器）超越 Teacher。
    3. Engineering Complexity (工程复杂度): 实现一套高效的 On-Policy Loop（生成 -> 缓存 -> 训练 -> 丢弃）比简单的 SFT DataLoader 要复杂得多。需要处理异步并发、分布式推理、显存碎片化等一系列系统级问题。

---

### 7. Insights & Spark (The "Soul")

* **Deep Reflection:**
  * On-Policy Distillation 暗示了一个深刻的技术趋势：静态数据集（Static Dataset）的重要性正在下降，动态环境（Dynamic Environment/Teacher）的重要性正在上升。
    * Past: AI 开发是 "Curate Dataset -> Train"。所有的知识都压缩在数据集中。
    * Future: AI 开发是 "Define Teacher/Verifier -> Student Interacts -> Feedback"。知识存在于 Teacher 的参数或 Verifier 的规则中，通过交互被 Student 吸收。这意味着我们的基建重点应该从“数据清洗流水线”转移到“高效的合成数据生成与评判流水线”。Compute is the new Data. 算力不仅用于训练，更用于生成“训练过程中的数据”。
  * Thinking Machines 的工作其实是 Self-Play (自我博弈) 的一种广义变体。
    * 如果我们将 Teacher 设定为 Student 本身（或者 Student 的一个旧版本），这不就是 OpenAI 的 Iterated Distillation 吗？
    * Idea: 我们可以实施 Iterative On-Policy Distillation。
      * Round 1: Student 1 蒸馏 Teacher (GPT-4)。
      * Round 2: Student 2 蒸馏 Student 1 (作为 Teacher)。
            ...
    * 这可能允许我们在没有持续访问 GPT-4 的情况下，通过 Student 的自我探索进一步提升性能（类似 AlphaZero 的自我博弈）。Student 在自我探索中生成的轨迹，通过自身的 Reward Model（如果有一个强 Reward Model）进行过滤，可以实现 Self-Improving。

* **Business Connection:**
  * 在对话或 Agent 场景中，传统的 SFT 需要离线收集数据、清洗、再训练，周期长（天级/周级），滞后且昂贵。
  * On-Policy Distillation 允许我们部署一个 "Shadow Mode (影子模式)" 的训练架构：
    1. Online Serving: 线上小模型（Student）处理实时用户流量。
    2. Shadow Evaluation: 离线大模型（Teacher）在后台对小模型的 Log（即 On-Policy 轨迹）进行打分和 Logits 回传。注意，这不需要实时完成，可以异步进行。
    3. Nightly Update: 利用这些累积的 Feedback，在夜间通过 On-Policy Distillation 更新小模型。
  * 这能实现 日更甚至小时级 的模型迭代，且由于有 Teacher 压阵，不会像纯 Online RL 那样容易因为 Reward Hacking 而跑飞，保证了业务的安全性。
