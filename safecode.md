# ParetoGen: 面向仓库级安全代码生成的安全性-可用性双目标选择

更新时间：2026-06-15

## 1. 重新定位：不是漏洞修复，而是安全代码生成

上一版把问题收敛成了 vulnerability repair，这样工程上更轻，但确实偏离了最开始讨论的方向。更准确的定位应该是：

```text
给定自然语言提示、漏洞相关代码上下文和仓库上下文，
让 LLM 生成缺失的安全关键代码片段。
```

这可以称为：

```text
repository-level secure code generation
contextual secure code generation
vulnerability-oriented code infilling
```

它和传统漏洞修复的区别是：

```text
漏洞修复:
  输入完整 vulnerable code
  输出 patch / diff

安全代码生成 / 补全:
  输入 prompt + repo context + masked code region
  输出 hole 中应该生成的代码
```

所以本文不主张“让模型修复已有漏洞”，而是研究：

```text
当模型在真实仓库上下文中生成安全敏感代码时，
如何选择既安全又可用的生成结果。
```

建议论文题目可以暂定为：

```text
ParetoGen:
Validation-guided Pareto Selection for Secure and Usable Repository-level Code Generation.
```

中文表述：

```text
ParetoGen: 面向仓库级安全代码生成的安全性-可用性双目标选择。
```

## 2. 为什么 A.S.E. 更适合作为主评测

你提到的 A.S.E. 方向是合适的。A.S.E. 全称是 **AI Code Generation Security Evaluation**，定位是 repository-level secure code generation benchmark。它的关键特点包括：

- 从真实仓库和 documented CVEs 构造任务。
- 保留完整仓库上下文，包括 build system 和跨文件依赖。
- 使用可复现的容器化评测框架。
- 提供对 security、build quality、generation stability 的评估。
- 比只在孤立函数片段上跑静态分析的 benchmark 更接近真实开发场景。

这正好适合本文的核心 insight：

```text
安全代码生成不是只让漏洞检测器闭嘴，
而是要在真实仓库中生成既安全又能集成、能构建、能保持功能语义的代码。
```

A.S.E.-style 任务可以理解为：

```text
从真实 CVE 相关仓库中定位安全关键代码区域；
删除或 mask 掉该区域；
给模型自然语言任务描述和仓库上下文；
要求模型生成缺失代码；
再用安全规则、构建结果和稳定性指标评估生成结果。
```

这比 Vul4J 更贴近“代码生成”。Vul4J / VJBench 更适合 vulnerability repair，即输入 vulnerable code、输出 patch。A.S.E. 则更自然地对应：

```text
masked secure code generation / infilling under repository context
```

因此建议：

```text
主评测: A.S.E.
补充讨论: RealSec-bench、SecRepoBench、CWEval。
Vul4J / VJBench: 放在 related work 中作为漏洞修复类 benchmark，而不是主线。
```

参考：

- A.S.E: <https://arxiv.org/abs/2508.18106>
- RealSec-bench: <https://arxiv.org/abs/2601.22706>
- SecRepoBench: <https://arxiv.org/abs/2504.21205>

## 3. 核心 idea

本文的核心观点可以压成一句话：

```text
LLM 安全代码生成不是单目标安全优化，
而是在安全性和可用性之间寻找 Pareto-optimal completion。
```

给定一个 A.S.E.-style 任务：

```text
x = prompt + repository context + masked code region
```

模型生成多个候选 completion：

```text
c_1, c_2, ..., c_N ~ pi(. | x)
```

每个候选 completion 都有至少两个目标：

```text
S(c): security score
U(c): usability / build-quality score
```

其中：

- `S(c)` 表示生成代码是否避免目标漏洞、是否满足安全规则、是否未触发高置信安全告警。
- `U(c)` 表示生成代码是否能放回仓库、是否能构建、是否通过已有测试或基本功能检查。

最重要的评价不是单独看 security，也不是单独看 build quality，而是：

```text
SUR = Secure-and-Usable Rate
    = P(S(c) = 1 and U(c) = 1)
```

也可以把 A.S.E. 的 generation stability 纳入第三个维度：

```text
G(c): generation stability / deterministic quality / formatting validity
```

但为了控制工作量，建议主论文只把 `S` 和 `U` 作为核心双目标，`G` 作为辅助指标。

## 4. 为什么这个问题值得做

现有安全代码生成研究常见的问题是：

```text
安全评测和功能评测分离。
```

例如：

- 在 SecurityEval / CyberSecEval / PurpleLlama 上测安全。
- 在 HumanEval / MBPP / EvalPlus 上测一般代码能力。

这样会产生一个问题：同一个生成结果是否同时安全且可用，并没有被充分验证。一个方法可能在安全 benchmark 上少了漏洞，但在真实仓库中无法构建；也可能在 HumanEval 上功能很好，但生成安全敏感代码时仍然引入漏洞。

近期一些工作已经指出类似现象：

- 安全提示或安全对齐可能导致功能下降、编译失败或生成无关代码。
- 只用静态分析器评估安全性会高估真实安全提升。
- 仓库级上下文下，生成代码需要满足跨文件依赖、构建系统、API 兼容和安全规则。

本文可以接着这个观察往前推进一步：

```text
既然安全性和可用性存在冲突，
那就不应该把候选 completion 简单压成一个单目标分数，
而应该把它们显式建模为 Pareto selection 问题。
```

## 5. 任务构造

### 5.1 输入

每个任务包含：

```text
natural language prompt
repository context
target file
masked code region
optional CVE / CWE / security hint
```

形式上：

```text
x = (I, R, F, H, Q)
```

其中：

- `I` 是自然语言指令。
- `R` 是仓库上下文，例如相关文件、依赖配置、调用关系。
- `F` 是目标文件。
- `H` 是被删除或 mask 的代码区域。
- `Q` 是可选安全提示，例如 CWE 类型、漏洞描述或安全规则。

### 5.2 输出

模型输出：

```text
c = generated code completion for the masked region
```

然后把 `c` 插回原仓库，形成候选仓库版本：

```text
R[c]
```

再运行 A.S.E. 或类似框架的 verifier：

```text
security verifier
build / test verifier
format / stability checker
```

### 5.3 Prompt 设计

prompt 不作为主贡献，只作为实验条件。

可以设计 4 类 prompt：

1. **Basic generation prompt**  
   只说明需要补全缺失代码。

2. **Security-aware prompt**  
   明确给出 CWE/CVE 或安全要求。

3. **Usability-aware prompt**  
   强调必须保持仓库构建、API 兼容和原有功能行为。

4. **Balanced prompt**  
   同时强调安全性和可用性。

示例：

```text
You are given a repository-level code generation task.
Fill in the masked region in the target file.
The generated code must integrate with the existing repository,
preserve the intended behavior, and avoid the security weakness described below.
Return only the code for the masked region.
```

这样可以研究：

```text
只强调安全是否会损害 build quality？
只强调可用性是否会保留漏洞模式？
balanced prompt 是否更容易生成 SU completion？
```

## 6. 安全性和可用性怎么定义

### 6.1 安全性 S

优先使用 A.S.E. 提供的 security evaluation：

```text
S(c) = 1 if generated completion satisfies expert-defined security rules
       and does not reintroduce the target vulnerability.
```

如果需要更细，可以分成：

```text
S_rule(c): 是否通过目标安全规则
S_static(c): 是否没有目标 CWE 的高置信静态分析告警
S_manual(c): 人工抽样是否确认没有明显漏洞
```

主实验不建议自己额外构造大规模 exploit tests。A.S.E. 的价值就是提供可复现、可审计的安全评估。自己补充人工 case study 即可。

### 6.2 可用性 U

可用性建议定义为：

```text
U(c) = 1 if generated completion can be inserted into the repository
       and the repository passes build-quality validation.
```

具体可以包括：

- 代码能插回 masked region。
- 格式和语法合法。
- 项目能编译 / 构建。
- 相关测试能通过。
- 生成代码没有破坏明显 API 约束。

如果 A.S.E. 已有 build quality score，就直接使用它作为 `U`。如果它是多级分数，可以二值化：

```text
U(c) = 1 if build quality >= threshold
```

也可以报告连续值：

```text
U_score(c) ∈ [0, 1]
```

### 6.3 稳定性 G

A.S.E. 还关注 generation stability。本文可以把它作为辅助维度：

```text
G(c): 多次生成或多次验证下结果是否稳定
```

但不建议把主问题扩成三目标优化。工程上可以这样处理：

```text
主目标: S + U
tie-breaker: G 或 minimality
```

## 7. 四象限分析

对每个候选 completion，标注：

```text
SU:   secure and usable
S¬U:  secure but unusable
¬SU:  usable but insecure
¬S¬U: insecure and unusable
```

四象限分析是本文最重要的 empirical insight。它可以回答：

- 安全代码生成失败主要是安全失败，还是可用性失败？
- 安全提示是否把候选从 `¬SU` 推到 `S¬U`，即安全变好但可用性变差？
- 可用性提示是否提升构建通过率，但仍然保留漏洞模式？
- 多候选采样中是否已经存在 `SU`，只是默认解码或评分策略没有选中？
- A.S.E. 的 security、build quality、stability 三类信号之间是否存在冲突？

这比只报告一个 average score 更能体现本文 insight。

## 8. Pareto 选择方法

### 8.1 候选生成

对每个任务和每种 prompt，采样多个候选：

```text
c_1, c_2, ..., c_N
```

建议第一阶段：

```text
N = 5 或 10
```

不要一开始做很大 N，因为仓库级验证成本较高。

### 8.2 候选评分

每个 completion 有：

```text
S(c) ∈ {0, 1}
U(c) ∈ {0, 1}
M(c): minimality / edit simplicity / length regularization
G(c): optional generation stability
```

其中 `M(c)` 不是核心目标，只作为 tie-breaker，避免选择过长、无关或过度复杂的生成。

简单定义：

```text
M(c) = - length(c)
```

或者：

```text
M(c) = - number_of_changed_lines_after_insertion
```

### 8.3 Pareto dominance

候选 `c_i` 支配 `c_j`，如果：

```text
S(c_i) >= S(c_j)
U(c_i) >= U(c_j)
```

且至少一个维度严格更好。

如果加入 `M` 或 `G`，建议只在 `S` 和 `U` 相同的时候使用它们，不要让它们压过安全性和可用性。

### 8.4 Selection rule

ParetoGen 的 deterministic selector：

```text
1. 运行 verifier，得到每个候选的 S 和 U。
2. 过滤出 Pareto frontier。
3. 如果存在 SU 候选，选择 minimality / stability 最好的 SU。
4. 如果不存在 SU，报告 failure，并记录最接近的失败类型:
   - only S¬U exists: 模型能安全但无法集成。
   - only ¬SU exists: 模型能集成但不安全。
   - only ¬S¬U exists: 两者都失败。
```

这个方法的重点不是复杂算法，而是评价逻辑：

```text
最终可接受的安全代码生成必须同时满足 S 和 U。
```

固定权重方法的问题是：

```text
score = alpha * S + beta * U
```

它会强行在 `S¬U` 和 `¬SU` 之间排序。但在安全代码生成里，这两类结果都不能作为最终可接受输出。ParetoGen 则把二者都视为未达成目标，并用四象限分析解释失败原因。

## 9. 可选扩展：Pareto-aware reranker

如果只做 deterministic selector，方法会比较轻，但可能被审稿人质疑技术贡献偏弱。因此可以准备一个可选增强：

```text
训练轻量 Pareto-aware reranker，而不是微调代码生成模型。
```

输入：

```text
task prompt
repository context summary
candidate completion
optional verifier summary
```

输出：

```text
rank score
```

训练 pair：

```text
SU  > S¬U
SU  > ¬SU
SU  > ¬S¬U
```

关键是：

```text
S¬U 和 ¬SU 不做强行全序排序。
```

因为二者分别代表不同失败模式。可以把它们都作为 rejected，但不要假设“安全但不可用”一定优于“可用但不安全”，或反过来。

这个 reranker 可以用自动 verifier 标签训练，不需要人工标注，也不需要功能性局部偏好数据。

## 10. 实验设计

### 10.1 主实验设置

建议主实验如下：

```text
Dataset: A.S.E.
Task: repository-level secure code generation / masked code infilling
Input: prompt + repo context + target file + masked region
Output: generated code completion
Models: 2-4 个 LLM
Sampling: 每个任务每种 prompt 采样 N 个 completion
Validation: A.S.E. security + build quality + stability evaluation
Selection: 比较不同 completion selection 策略
```

模型选择：

- 一个强闭源模型，例如 GPT-4o / Claude / DeepSeek API。
- 一个强开源代码模型，例如 Qwen2.5-Coder / DeepSeek-Coder / CodeLlama。
- 如果预算有限，优先多跑任务，少跑模型。

采样设置：

```text
temperature = 0.2 / 0.7
N = 5 或 10
```

建议不要一开始测试过多模型。本文更需要证明 Pareto 评估和选择的价值，而不是追求模型 leaderboard。

### 10.2 Selection baselines

至少比较这些选择策略：

1. **First sample**  
   直接取第一个 completion。

2. **Security-only selection**  
   优先选择 `S=1` 的 completion，不考虑 `U`。

3. **Usability-only selection**  
   优先选择 `U=1` 的 completion，不考虑 `S`。

4. **Scalar weighted selection**  
   使用固定权重：

   ```text
   score = alpha * S + beta * U + gamma * M
   ```

5. **ParetoGen selection**  
   优先选择 Pareto frontier 上的 `SU` completion。

6. **Oracle Any-SUR@N**  
   只要 N 个候选中存在任意 `SU`，就算成功。这个是 best-of-N 上界。

如果做 reranker，再加：

7. **Pareto-aware reranker**  
   用训练集中的 verifier 标签训练，测试集上选择 completion。

### 10.3 Prompt baselines

prompt 作为 ablation：

- Basic prompt。
- Security-aware prompt。
- Usability-aware prompt。
- Balanced prompt。

重点观察：

```text
Security-aware prompt 是否提高 S 但降低 U？
Usability-aware prompt 是否提高 U 但降低 S？
Balanced prompt 是否提高 SU？
ParetoGen 是否比 first sample 更接近 Oracle Any-SUR@N？
```

### 10.4 指标

主指标：

```text
SUR@1 = selected completion satisfies S=1 and U=1
```

辅助指标：

- `Security@1`：选中 completion 是否安全。
- `Usability@1`：选中 completion 是否可用。
- `Any-SUR@N`：N 个候选中是否存在任意 `SU`。
- `S¬U Rate`：安全但不可用的候选比例。
- `¬SU Rate`：可用但不安全的候选比例。
- `¬S¬U Rate`：安全和可用性都失败的候选比例。
- `Pareto Frontier Size`：每个任务的非支配候选数量。
- `Gap to Oracle`：`Any-SUR@N - SUR@1`。
- `Validation Cost`：平均每个任务验证多少候选。
- `Stability`：沿用 A.S.E. 的 generation stability 指标。

可以定义一个 conflict rate：

```text
Conflict Rate = P(exists S¬U or ¬SU among candidates)
```

或者更强：

```text
Hard Conflict Rate = P(exists S¬U and exists ¬SU but no SU)
```

这个指标能说明安全性和可用性之间的冲突不是想象出来的。

### 10.5 Research Questions

建议设置 4 个 RQ。

**RQ1: 仓库级安全代码生成中，安全性和可用性冲突有多明显？**

报告所有候选 completion 的四象限分布：

```text
SU / S¬U / ¬SU / ¬S¬U
```

预期观察：

```text
不少候选不是简单失败，而是安全和可用性只满足一边。
```

**RQ2: ParetoGen selection 是否优于 first sample、security-only、usability-only 和 scalar selection？**

比较：

```text
SUR@1
Security@1
Usability@1
Gap to Oracle
```

预期观察：

```text
security-only 可能提升 S 但牺牲 U；
usability-only 可能提升 U 但保留漏洞；
scalar selection 对权重敏感；
ParetoGen 在 SUR@1 上更稳。
```

**RQ3: best-of-N 中是否已经存在安全且可用的 completion？**

比较：

```text
First sample SUR@1
ParetoGen SUR@1
Oracle Any-SUR@N
```

如果 `Any-SUR@N` 明显高于 first sample，说明：

```text
模型并非完全不会生成安全可用代码，关键问题之一是 selection。
```

**RQ4: 不同 prompt 如何影响四象限分布？**

比较 basic、security-aware、usability-aware、balanced prompt。

预期观察：

```text
安全提示不一定提升最终 SUR；
它可能把一部分 ¬SU 变成 S¬U。
balanced prompt + Pareto selection 更可能得到 SU。
```

### 10.6 可选实验

如果主实验顺利，可以加两个小实验。

第一，跨 benchmark 小规模验证：

```text
在 RealSec-bench 或 SecRepoBench 子集上复现四象限分析。
```

这能说明 ParetoGen 不是只适配 A.S.E.。

第二，轻量 reranker：

```text
用 A.S.E. 训练 split 中的候选 completion 和 verifier 标签训练 reranker；
在 held-out tasks 上选择 completion；
比较 ParetoGen deterministic selector 和 learned reranker。
```

这个扩展可以作为增强版方法，但不是 MVP 必需。

## 11. 为什么不做功能性局部偏好

功能性局部偏好数据仍然不建议作为主线。

在 A.S.E.-style 任务中，安全关键区域是 masked region，本身已经把局部性引入了任务。模型只需要生成 hole 内代码，不需要在完整文件里定位哪里该改。

而功能性局部偏好依然难构造：

- build failure 可能来自 import、类型、配置、跨文件 API。
- test failure 不一定能定位到 completion 中哪一行。
- 仓库级上下文下，功能失败可能是跨文件交互导致的。
- 依赖 coverage / slicing / fault localization 会显著增加工程量。

因此建议：

```text
不做 utility-local preference。
把 utility 作为 completion-level constraint。
```

本文的核心不是 token-level credit assignment，而是：

```text
在安全代码生成中，最终可接受结果必须同时满足安全和可用。
```

这正好和 A.S.E. 的 repository-level verifier 匹配。

## 12. 和现有工作的差异

### 12.1 相比 SecurityEval / CyberSecEval / PurpleLlama

这些 benchmark 常用于评估生成代码是否包含 CWE 风险，但很多任务是孤立片段，功能验证弱，和真实仓库构建集成距离较远。

本文使用 A.S.E.-style 仓库级任务，强调：

```text
same generated code, same repository, jointly evaluated for security and usability.
```

### 12.2 相比 RealSec-bench / SecRepoBench

RealSec-bench 和 SecRepoBench 已经强调真实仓库和安全-功能共同评估。本文的差异可以放在：

```text
不是提出一个新 benchmark，
而是提出一个 Pareto selection / analysis protocol，
系统揭示多候选安全代码生成中的 S-U tradeoff。
```

### 12.3 相比 Vul4J / VJBench

Vul4J / VJBench 更偏漏洞修复：

```text
vulnerable code -> patch
```

本文聚焦安全代码生成 / 补全：

```text
prompt + context + masked region -> generated code
```

可以在 related work 中承认二者相关，但不要把本文主线写成 repair。

## 13. 论文贡献可以怎么写

建议贡献点压成三个：

1. **Problem framing**  
   将仓库级安全代码生成形式化为安全性与可用性的双目标 Pareto selection 问题，指出单独报告安全分或构建分会掩盖真实失败模式。

2. **Method**  
   提出 ParetoGen，一个基于现成 repository-level verifier 的 best-of-N completion selection 方法，显式区分 `SU`、`S¬U`、`¬SU`、`¬S¬U` 四类候选。

3. **Empirical study**  
   在 A.S.E. 等仓库级安全代码生成 benchmark 上，系统分析不同模型、prompt 和 selection 策略的四象限分布，证明 Pareto selection 能提高 `SUR@1` 并减少安全性-可用性 tradeoff 被单指标掩盖的问题。

如果做了 reranker，可以加第四点：

4. **Optional learned selection**  
   使用自动 verifier 标签训练轻量 Pareto-aware reranker，在不微调代码生成模型、不构造人工局部偏好的情况下进一步缩小到 Oracle@N 的差距。

## 14. 最小可行工作量

一个学生可执行的 MVP：

```text
1. 跑通 A.S.E. benchmark 的一小批任务。
2. 为每个任务构造 3-4 种 prompt。
3. 每种 prompt 采样 5 个 completion。
4. 把 completion 插回 masked region。
5. 运行 A.S.E. security / build quality / stability verifier。
6. 记录每个 completion 的 S/U/G 标签。
7. 实现 first sample、security-only、usability-only、scalar、ParetoGen 五类 selection。
8. 报告 SUR@1、Any-SUR@N、四象限分布、Gap to Oracle。
9. 人工分析 8-10 个代表性 case。
```

不需要：

- 构建新 benchmark。
- 自己写大量功能测试。
- 做复杂静态分析。
- 训练大模型。
- 做功能性局部偏好数据。
- 从漏洞修复数据集里重新设计任务。

如果时间充足：

```text
10. 加 RealSec-bench / SecRepoBench 小子集。
11. 加轻量 Pareto-aware reranker。
```

## 15. 摘要草稿

```text
Large language models are increasingly used to generate security-critical code in real-world repositories. Existing secure code generation evaluations often report security and usability separately, which can obscure an important failure mode: generated code may avoid the target vulnerability while failing to build or integrate, or it may build successfully while reintroducing the vulnerable pattern. We formulate repository-level secure code generation as a bi-objective Pareto selection problem over security and usability. We propose ParetoGen, a validation-guided best-of-N selection framework for A.S.E.-style masked code generation tasks. ParetoGen samples multiple completions for each masked security-critical region, evaluates them with repository-level security and build-quality verifiers, classifies them into secure-and-usable, secure-but-unusable, insecure-but-usable, and insecure-and-unusable quadrants, and selects Pareto-optimal completions that satisfy both objectives. Experiments on A.S.E. compare first-sample decoding, security-only selection, usability-only selection, scalar weighted selection, and Pareto selection across multiple prompting strategies. Results are reported using Secure-and-Usable Rate, Any-SUR@N, quadrant distributions, and gap to oracle selection, revealing the tradeoff between vulnerability avoidance and repository-level usability in LLM-generated code.
```

## 16. 一句话 selling point

```text
安全代码生成不是选择“最安全”的 completion，
也不是选择“最容易构建”的 completion，
而是从多个候选中找到同时安全且可用的 Pareto-optimal completion。
```

## 17. Prism / AAAI LaTeX 写作模板

下面这段可以作为 AAAI LaTeX 模板中 `\begin{document}` 和 `\end{document}` 之间的主体内容。为了方便 Prism 扩写，所有章节都先用 `\subsection` 和 `\subsubsection`，每个部分用中文注释写清楚写作要求。正式投稿时，可以根据 AAAI 模板习惯把最外层 `\subsection` 改成 `\section`。

```latex
% ===================== Prism Writing Template =====================
% 使用说明：
% 1. 将本段粘贴到 AAAI LaTeX 模板的 \begin{document} 之后。
% 2. 如果模板中已有 \title、\author、\maketitle，请只复制 \begin{abstract} 之后的正文。
% 3. 请 Prism 将所有中文提示扩写为英文论文正文，并删除提示性注释。
% 4. 全文主线必须围绕 ParetoGen: validation-guided Pareto selection for secure and usable repository-level code generation。
% 5. 不要把本文写成 vulnerability repair 论文；本文任务是 A.S.E.-style masked secure code generation / code infilling。

\begin{abstract}
% Prism 提示：
% 请写一段 AAAI 风格英文摘要，长度控制在 180--230 词左右。
% 摘要需要包含以下要点：
% 1. 背景：LLM 正在被用于生成真实仓库中的安全敏感代码。
% 2. 问题：现有安全代码生成评测常把 security 和 usability/build quality 分开报告，掩盖了安全但不可用、可用但不安全这两类失败。
% 3. 方法：提出 ParetoGen，一个 validation-guided best-of-N selection framework。
% 4. 设定：在 A.S.E.-style masked repository-level secure code generation tasks 上，对多个 completion 运行 security 和 build-quality verifier。
% 5. 核心机制：将候选分为 SU、S¬U、¬SU、¬S¬U 四象限，并选择同时满足安全和可用的 Pareto-optimal completion。
% 6. 指标：Secure-and-Usable Rate, Any-SUR@N, quadrant distribution, gap to oracle。
% 7. 结论：Pareto-aware selection 更准确刻画并缓解安全性与可用性的 tradeoff。
\end{abstract}

\subsection{Introduction}

\subsubsection{Motivation}
% Prism 提示：
% 请写引言开头，强调 LLM 已经从生成孤立函数转向生成真实仓库中的安全敏感代码。
% 举例说明仓库级代码生成需要同时满足安全规则、编译构建、跨文件依赖、API 兼容和行为保持。
% 语气要像 AI/SE/Security 交叉论文，避免夸张表述。

\subsubsection{Problem: Security and Usability Are Often Evaluated Separately}
% Prism 提示：
% 请指出现有 secure code generation 评测常见做法：
% 1. 在 SecurityEval、CyberSecEval、PurpleLlama 等安全 benchmark 上测漏洞风险；
% 2. 在 HumanEval、MBPP、EvalPlus 等一般代码任务上测功能能力；
% 3. 但同一个生成结果是否同时安全且可用，往往没有被充分评估。
% 请解释这会导致误判：一个方法可能降低安全告警，但生成代码无法构建；也可能构建成功但仍保留漏洞模式。

\subsubsection{Key Insight}
% Prism 提示：
% 请清楚提出本文 insight：
% repository-level secure code generation should be treated as a bi-objective selection problem over security and usability.
% 不要把 security 和 usability 简单压成单一分数。
% 最终可接受的 completion 必须同时满足 S=1 和 U=1。
% 强调本文不是研究漏洞修复，而是研究 masked security-critical code generation / code infilling。

\subsubsection{Our Approach: ParetoGen}
% Prism 提示：
% 请简要介绍 ParetoGen：
% 1. 对每个 A.S.E.-style task 采样多个 candidate completions；
% 2. 将 completion 插回仓库；
% 3. 运行 repository-level security verifier 和 build-quality/usability verifier；
% 4. 将候选分为 SU、S¬U、¬SU、¬S¬U；
% 5. 用 Pareto dominance 和 minimality/stability tie-breaker 选择 completion。
% 需要强调 ParetoGen 是轻量方法，不需要训练大模型，也不需要构造新测试。

\subsubsection{Contributions}
% Prism 提示：
% 请写 3--4 条贡献，使用英文 bullet list。
% 建议贡献如下：
% 1. We formulate repository-level secure code generation as a bi-objective Pareto selection problem.
% 2. We propose ParetoGen, a validation-guided best-of-N completion selection framework.
% 3. We introduce / use Secure-and-Usable Rate and quadrant-based analysis to reveal S-U tradeoffs.
% 4. We conduct experiments on A.S.E.-style benchmark tasks comparing first-sample, security-only, usability-only, scalar weighted, and Pareto selection.

\subsection{Related Work}

\subsubsection{Secure Code Generation Benchmarks}
% Prism 提示：
% 请综述 SecurityEval、CyberSecEval/PurpleLlama、CWEval 等安全代码生成评测。
% 重点不是逐篇详细介绍，而是指出它们推动了安全代码生成评估，但很多设定是片段级或安全/功能分离。
% 请自然过渡到 repository-level benchmark 的必要性。

\subsubsection{Repository-Level Secure Code Generation}
% Prism 提示：
% 请介绍 A.S.E.、RealSec-bench、SecRepoBench 等仓库级安全代码生成评测。
% 强调它们比孤立函数任务更接近真实开发，因为保留了 build system、依赖、上下文和跨文件约束。
% 请说明本文不提出新 benchmark，而是在此类 benchmark 上提出 Pareto selection / analysis protocol。

\subsubsection{LLM Vulnerability Repair}
% Prism 提示：
% 请简要介绍 Vul4J、VJBench 等漏洞修复类 benchmark。
% 明确区分：
% vulnerability repair: vulnerable code -> patch；
% secure code generation / infilling: prompt + context + masked region -> generated code。
% 说明本文和漏洞修复相关，但主任务不是 patch generation。

\subsubsection{Multi-Objective Optimization and Selection for Code Generation}
% Prism 提示：
% 请综述多目标代码生成、best-of-N sampling、reranking、pass@k / oracle@k 思路。
% 引出本文的差异：不是只按测试通过率或安全分选，而是显式使用 security-usability Pareto criterion。

\subsection{Problem Formulation}

\subsubsection{Task Definition}
% Prism 提示：
% 请形式化 A.S.E.-style repository-level secure code generation task。
% 定义输入 x = (I, R, F, H, Q)，其中：
% I 是自然语言指令；
% R 是 repository context；
% F 是 target file；
% H 是 masked security-critical region；
% Q 是可选 CVE/CWE/security hint。
% 模型输出 completion c，并将其插回仓库得到 R[c]。

\subsubsection{Security Objective}
% Prism 提示：
% 请定义 security verifier S(c)。
% S(c)=1 表示 completion 满足目标安全规则、没有重新引入目标漏洞或高置信安全告警。
% 如果 A.S.E. 提供 security score，可以把二值和连续形式都写出来：
% S(c) in {0,1} 或 S_score(c) in [0,1]。

\subsubsection{Usability Objective}
% Prism 提示：
% 请定义 usability/build-quality verifier U(c)。
% U(c)=1 表示 completion 可以插回仓库、语法合法、构建通过、相关测试或集成检查通过。
% 强调 usability 不是泛泛的用户满意度，而是 repository-level build and integration quality。

\subsubsection{Secure-and-Usable Rate}
% Prism 提示：
% 请定义主指标：
% SUR@1 = P(S(c_selected)=1 and U(c_selected)=1)。
% 定义 Any-SUR@N = N 个候选中是否存在任意 S=1 且 U=1 的 completion。
% 解释 Any-SUR@N 是 best-of-N oracle upper bound，用于判断模型是否已经生成了好 completion 但 selection 没选中。

\subsubsection{Four-Quadrant Candidate Taxonomy}
% Prism 提示：
% 请定义四类候选：
% SU: secure and usable；
% S¬U: secure but unusable；
% ¬SU: usable but insecure；
% ¬S¬U: insecure and unusable。
% 解释四象限分析为什么重要：它能揭示安全代码生成失败到底是安全失败、可用性失败，还是二者冲突。

\subsection{Method: ParetoGen}

\subsubsection{Overview}
% Prism 提示：
% 请用一段话概述 ParetoGen pipeline。
% Pipeline 包括 candidate generation, repository insertion, validation, quadrant classification, Pareto selection。
% 可以引用一个 Algorithm 环境，但如果没有具体伪代码，也可以用文字清晰描述。

\subsubsection{Candidate Completion Generation}
% Prism 提示：
% 请描述如何对每个 task 采样 N 个 completion。
% 说明可以改变 temperature 或 prompt variant 来增加候选多样性。
% 推荐写 N=5 或 N=10，避免工程成本过高。
% 强调本文关注 selection protocol，不追求最大规模采样。

\subsubsection{Repository-Level Validation}
% Prism 提示：
% 请描述每个 completion 如何插回 masked region，并运行 A.S.E. verifier。
% 验证包括 security evaluation、build-quality evaluation、generation stability 或 formatting checks。
% 说明所有标签来自自动 verifier，因此不需要人工构造大规模功能测试。

\subsubsection{Pareto Dominance}
% Prism 提示：
% 请形式化 Pareto dominance：
% c_i dominates c_j if S(c_i) >= S(c_j), U(c_i) >= U(c_j), and at least one is strictly better。
% 如果 S 和 U 都是二值，说明 SU 支配其他三类；S¬U 和 ¬SU 互不支配。
% 解释为什么不要强行认为 S¬U 优于 ¬SU 或反过来。

\subsubsection{Selection Rule}
% Prism 提示：
% 请描述 deterministic ParetoGen selector：
% 1. 计算所有候选的 S 和 U；
% 2. 找 Pareto frontier；
% 3. 如果存在 SU，选择 minimality 或 stability 最好的 SU；
% 4. 如果不存在 SU，报告 failure type，并用于分析。
% 请说明 minimality/stability 只是 tie-breaker，不能压过 S 和 U。

\subsubsection{Optional Pareto-Aware Reranker}
% Prism 提示：
% 这一节写成可选扩展。
% 说明可以用自动 verifier 标签训练轻量 reranker，而不是微调 LLM。
% 训练偏好包括 SU > S¬U, SU > ¬SU, SU > ¬S¬U。
% 明确不对 S¬U 和 ¬SU 做全序排序，因为它们代表不同失败模式。
% 如果论文实验没做 reranker，就把这一节改成 Discussion 或 Future Extension。

\subsection{Experimental Setup}

\subsubsection{Dataset}
% Prism 提示：
% 请写主评测使用 A.S.E.。
% 说明 A.S.E. 是 repository-level secure code generation benchmark，任务来自真实仓库和 documented CVEs，并提供安全、构建质量、稳定性评估。
% 如果实际实验只跑了 A.S.E. 子集，请说明选择标准，例如可复现、构建成功、成本可控。

\subsubsection{Models}
% Prism 提示：
% 请描述实验模型。
% 至少包括一个强闭源模型和一个强开源代码模型。
% 模型名先用占位符，例如 GPT-4o / Claude / DeepSeek / Qwen2.5-Coder，最终根据实际实验替换。
% 不要编造模型结果。

\subsubsection{Prompt Variants}
% Prism 提示：
% 请定义 4 个 prompt variants：
% Basic prompt: 只要求补全 masked code；
% Security-aware prompt: 加入 CWE/CVE 或安全规则；
% Usability-aware prompt: 强调构建、API 兼容和行为保持；
% Balanced prompt: 同时强调安全和可用。
% 说明 prompt 不是主贡献，而是用于研究不同指令如何影响四象限分布。

\subsubsection{Baselines}
% Prism 提示：
% 请描述 selection baselines：
% First sample；
% Security-only selection；
% Usability-only selection；
% Scalar weighted selection；
% ParetoGen selection；
% Oracle Any-SUR@N。
% 说明 Oracle Any-SUR@N 是上界，不是实际部署方法。

\subsubsection{Metrics}
% Prism 提示：
% 请列出并解释指标：
% SUR@1；
% Security@1；
% Usability@1；
% Any-SUR@N；
% S¬U Rate；
% ¬SU Rate；
% ¬S¬U Rate；
% Gap to Oracle；
% Pareto Frontier Size；
% Validation Cost；
% Stability。
% 强调 SUR@1 是主指标。

\subsection{Results}

\subsubsection{RQ1: How Severe Is the Security-Usability Conflict?}
% Prism 提示：
% 请写 RQ1 的结果段落。
% 需要报告四象限分布，解释有多少候选落在 S¬U 和 ¬SU。
% 如果实验数据还没有填入，请使用占位符表格引用，例如 Table~\ref{tab:quadrants}。
% 不要编造具体数字；用 [XX] 作为待填结果。

\subsubsection{RQ2: Does ParetoGen Improve SUR@1?}
% Prism 提示：
% 请比较 First sample、Security-only、Usability-only、Scalar weighted、ParetoGen。
% 重点解释 ParetoGen 是否提高 SUR@1，是否避免 security-only 或 usability-only 的偏置。
% 使用 Table~\ref{tab:selection} 作为占位符。

\subsubsection{RQ3: How Large Is the Gap to Oracle Any-SUR@N?}
% Prism 提示：
% 请比较 First sample SUR@1、ParetoGen SUR@1 和 Oracle Any-SUR@N。
% 解释如果 Any-SUR@N 明显高于 First sample，说明模型已经能生成好 completion，但 selection 很关键。
% 使用 Figure~\ref{fig:oracle-gap} 作为占位符。

\subsubsection{RQ4: How Do Prompt Variants Affect the Pareto Frontier?}
% Prism 提示：
% 请分析 Basic、Security-aware、Usability-aware、Balanced prompt。
% 重点讨论 security-aware prompt 是否可能增加 S¬U，usability-aware prompt 是否可能增加 ¬SU，balanced prompt 是否提高 SU。
% 使用 Figure~\ref{fig:prompt-quadrants} 作为占位符。

\subsection{Analysis}

\subsubsection{Case Study: Secure but Unusable Completion}
% Prism 提示：
% 请写一个案例分析，展示模型生成了安全但无法构建或无法集成的代码。
% 解释失败原因可能是缺失 import、API 不兼容、类型错误、删除必要逻辑或生成过度保守检查。
% 如果没有真实案例数据，请保留为占位符，不要编造仓库名。

\subsubsection{Case Study: Usable but Insecure Completion}
% Prism 提示：
% 请写一个案例分析，展示模型生成的代码可以构建，但仍然保留目标漏洞模式。
% 解释这类失败为什么单看 build quality 会被误判为成功。

\subsubsection{Why Scalar Selection Can Be Misleading}
% Prism 提示：
% 请分析固定权重 score = alpha S + beta U 的问题。
% 说明不同 alpha/beta 可能导致选择 S¬U 或 ¬SU，但二者都不是最终可接受结果。
% 强调 ParetoGen 的优势是将 S 和 U 作为硬目标，而不是过早压成单分数。

\subsubsection{Validation Cost}
% Prism 提示：
% 请讨论 best-of-N validation 的成本。
% 说明 N=5 或 N=10 在仓库级任务中是较现实的折中。
% 如果有实际运行时间，填入平均验证时间；否则留 [XX]。

\subsection{Threats to Validity}

\subsubsection{Benchmark Scope}
% Prism 提示：
% 请说明结果主要基于 A.S.E. 或其子集，可能不覆盖所有语言、CWE 类型和仓库生态。
% 说明未来可以在 RealSec-bench、SecRepoBench 或更多仓库级 benchmark 上验证。

\subsubsection{Verifier Limitations}
% Prism 提示：
% 请承认 security verifier 和 build-quality verifier 都不完美。
% 通过安全规则不等于完全无漏洞，构建通过也不等于完整语义正确。
% 但 repository-level verifier 比孤立静态告警更接近真实评估。

\subsubsection{Sampling and Model Dependence}
% Prism 提示：
% 请说明 best-of-N 结果依赖采样温度、N、模型能力和 prompt。
% 论文应报告这些设置，并避免过度泛化。

\subsubsection{Human Evaluation}
% Prism 提示：
% 如果进行了人工 case study，请说明人工检查规模有限。
% 如果没有人工评估，请说明本文主要依赖自动 verifier，并把人工验证作为未来工作。

\subsection{Discussion}

\subsubsection{Implications for Secure Code Generation Evaluation}
% Prism 提示：
% 请讨论本文对安全代码生成评测的启示：
% 未来 benchmark 不应只报告 security pass rate 或 build pass rate，而应报告 joint secure-and-usable metrics。

\subsubsection{When to Use ParetoGen}
% Prism 提示：
% 请说明 ParetoGen 适用于能自动验证多个候选的场景，例如 IDE code generation、CI-based code assistant、agentic coding workflow。
% 不适用于无法运行 verifier 或验证成本极高的场景。

\subsubsection{Future Work}
% Prism 提示：
% 请提出未来方向：
% 1. learned Pareto-aware reranker；
% 2. active sampling for diverse completions；
% 3. combining dynamic exploit tests with build validation；
% 4. extending to multi-file generation and agentic coding。

\subsection{Conclusion}
% Prism 提示：
% 请写简洁结论。
% 重申本文将仓库级安全代码生成建模为安全性和可用性的双目标 Pareto selection 问题。
% 总结 ParetoGen 的做法和主要发现。
% 结尾强调：安全代码生成的目标不是最安全或最容易构建的单方面结果，而是同时安全且可用的 completion。

\subsection{Ethical Considerations}
% Prism 提示：
% 请写 AAAI 风格伦理声明。
% 说明本文目标是提升 LLM 生成安全代码的能力，降低漏洞引入风险。
% 数据来自公开 benchmark，不发布新的漏洞利用代码。
% 讨论潜在风险：攻击者可能学习安全评估流程；缓解方式是只报告防御性方法和聚合结果。

\subsection{Reproducibility Checklist}
% Prism 提示：
% 请写一个简短 reproducibility checklist。
% 包含 benchmark 版本、模型名称、prompt 模板、采样参数、N、verifier 配置、selection 代码、随机种子和硬件/运行环境。

% ===================== Optional Table / Figure Placeholders =====================

\subsection{Placeholder Tables and Figures}

\subsubsection{Table: Quadrant Distribution}
% Prism 提示：
% 请在正式论文中添加 Table~\ref{tab:quadrants}。
% 表格列建议：
% Model, Prompt, SU Rate, S¬U Rate, ¬SU Rate, ¬S¬U Rate。

\subsubsection{Table: Selection Results}
% Prism 提示：
% 请在正式论文中添加 Table~\ref{tab:selection}。
% 表格列建议：
% Method, SUR@1, Security@1, Usability@1, Gap to Oracle, Validation Cost。

\subsubsection{Figure: Gap to Oracle}
% Prism 提示：
% 请在正式论文中添加 Figure~\ref{fig:oracle-gap}。
% 展示 First sample、ParetoGen、Oracle Any-SUR@N 的差距。

\subsubsection{Figure: Prompt Effects}
% Prism 提示：
% 请在正式论文中添加 Figure~\ref{fig:prompt-quadrants}。
% 展示不同 prompt 的四象限分布。
```
