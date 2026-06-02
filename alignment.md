# Agent 安全对齐与 LLM 安全代码生成对齐调研笔记

更新时间：2026-06-02

本文整理两条相关研究线：

1. 面向 LLM Agent 的 RL / 偏好优化安全对齐。
2. 面向 LLM 安全代码生成的 RL / DPO / 偏好优化对齐。

核心判断是：这两个方向都在从“单轮回答是否安全”转向“轨迹级、工具感知、verifier 驱动”的安全对齐。Agent 场景里，风险常常不是最终回答中的一句话，而是中间某次工具调用、文件操作、记忆写入、外部动作或对恶意 skill 的服从。代码生成场景里，漏洞也经常只集中在少数几行或少数 token，而不是整段代码都错。

## 1. Agent 安全对齐

### 1.1 问题转变

传统 LLM 安全对齐主要看模型面对 harmful prompt 时是拒绝还是服从。Agent 安全更复杂，因为模型会和环境交互：

- 读取本地 skills、文件、邮件、网页、记忆、日志。
- 调用 shell、浏览器、代码执行、邮件、文件编辑等工具。
- 在最终回答之前就可能产生真实副作用。
- 有害行为可能隐藏在中间轨迹，而不是最终输出。

因此，Agent 安全不能只评估最终回答，而应该评估完整轨迹：

```text
state -> observation/context -> planning/reasoning -> action/tool call -> environment transition -> final artifact
```

一个安全 Agent 需要同时做到：

- 完成正常用户任务。
- 抵抗 prompt injection。
- 不越权、不扩大任务范围。
- 不泄露数据。
- 不执行危险工具调用。
- 不植入持久化后门或污染记忆。
- 不因为过度保守而拒绝正常任务。

### 1.2 Skill 和 workspace 成为新的攻击面

现代 Agent 不只看用户 prompt，还会读取本地 skill、helper script、模板、sidecar file、本地 corpus、缓存、记忆和工具 wrapper。这些内容对 Agent 来说通常像“可信上下文”，因此形成了类似供应链的攻击面。

代表性工作：

- **SkillSafetyBench**：把攻击放在 skill-facing context 中，同时保持用户任务本身是正常任务。它区分 `task_success` 和 `attack_success`，包含 155 个 cases、6 个 risk domains、30 个 categories。  
  <https://arxiv.org/abs/2605.12015>

- **Skill-Inject**：研究 skill 文件中的 prompt injection，指出通过 skill 注入可以诱导 Agent 产生数据外泄、破坏性操作等行为。  
  <https://arxiv.org/abs/2602.20156>

- **"Do Not Mention This to the User"**：从真实 skill 生态中测量恶意 skills，强调 skills 已经成为 Agent 供应链安全风险。  
  <https://arxiv.org/abs/2602.06547>

- **HarmfulSkillBench**：研究 harmful skills，以及预安装 skill 如何降低模型对 harmful task 的拒绝率。  
  <https://arxiv.org/abs/2604.15415>

- **ClawSafety**：展示 safety-aligned LLM 一旦嵌入高权限本地 Agent，并接触 skill、邮件、网页等上下文，仍可能产生高 attack success rate。  
  <https://arxiv.org/abs/2604.01438>

这一类工作共同说明：Agent 安全不只是模型 alignment 问题，也是上下文信任、工具权限、skill 供应链和 workspace artifact 安全问题。

## 2. 基于 RL 的 Agent 安全对齐

### 2.1 与 AgenticRL 的关系

基于强化学习的 Agent 安全对齐和 AgenticRL 高度重合。区别主要在 reward 设计：

```text
普通 AgenticRL:
  maximize task_success

Agent 安全对齐:
  maximize task_success
  minimize attack_success
  minimize unsafe_tool_call
  minimize data_boundary_violation
  minimize over_refusal
```

也可以写成约束优化：

```text
maximize task_success
subject to harmful_action_rate <= epsilon
           data_leakage_rate <= epsilon
           over_refusal_rate <= epsilon
```

所以一个简洁定义是：

```text
Agent 安全对齐 = AgenticRL
              + safety verifier
              + trajectory-level reward model
              + constrained / Pareto optimization
```

### 2.2 近期关键工作

- **FATE: On-Policy Self-Evolution via Failure Trajectories for Agentic Safety Alignment**  
  这是非常直接的 Agent 安全对齐工作。它把 verifier 打分的失败轨迹转换成 repair supervision，再用 Pareto-style policy optimization 同时优化 safety、utility、trajectory validity 和 over-refusal。它的核心观点是：response-level、off-policy、single sparse reward 都不足以解决 Agent 安全问题。  
  <https://arxiv.org/abs/2605.11882>

- **AgentDoG 1.5**  
  更偏框架和生态。它提供 Agent safety taxonomy、benchmark settings、安全感知的 SFT/RL 环境，以及轻量 guardrail。对“如何定义 Agent 安全 reward、如何构建训练环境”很有参考价值。  
  <https://arxiv.org/abs/2605.29801>

- **Agents of Chaos**  
  在带有 persistent memory、email、Discord、filesystem、shell 的 live lab 中红队 autonomous agents，展示数据泄露、未授权动作、破坏性操作、资源消耗、跨 Agent 传播等风险。  
  <https://arxiv.org/abs/2602.20021>

- **From Helpfulness to Toxic Proactivity**  
  研究 Agent 为了“有用”而主动采取不道德、操纵性或越界行为的现象。它强调多轮轨迹中的 misalignment，而不是单轮 harmful prompt。  
  <https://arxiv.org/abs/2602.04197>

- **T-MAP / A3S-Bench**  
  这两类工作关注 trajectory-aware red-teaming、多轮 evasive attack、跨时间和跨空间隐藏攻击意图。它们对构造安全 RL 训练环境和 adversarial evaluation 有启发。  
  <https://arxiv.org/abs/2603.22341>  
  <https://arxiv.org/abs/2605.22321>

## 3. 稀疏奖励与轨迹级 credit assignment

Agent 安全 reward 通常是稀疏、延迟的。很多 benchmark 只在最后给出：

```text
task_success: true/false
attack_success: true/false
over_refusal: true/false
```

但训练时真正需要知道的是：哪一步导致了安全失败？

可能的责任点包括：

- 读取了被污染 skill 后没有识别不可信指令。
- 调用了超出用户授权范围的工具。
- 写入了持久化 hook 或后门文件。
- 把数据输出到未授权目的地。
- 把攻击指令写入长期记忆。
- 为了避免风险直接拒绝了正常任务。

相关 AgenticRL 工作：

- **GiGPO**  
  处理 multi-turn Agent 的 sparse / delayed reward 问题，提出 episode-level macro advantage 和 step-level micro advantage，用于更细粒度 credit assignment。  
  <https://arxiv.org/abs/2505.10978>

- **AEM**  
  在只有 outcome reward 的情况下，用 response-level entropy dynamics 做 supervision-free credit assignment。对安全场景的启发是：即使 verifier 只给最终 binary label，也可以利用不确定性和探索信号改善训练。  
  <https://arxiv.org/abs/2605.00425>

- **RAGEN / RAGEN-2**  
  研究 reasoning agents 的 RL 训练，指出在 reward 信号弱、低方差或不够细粒度时，模型容易学到 shallow strategies 或 template collapse。  
  <https://arxiv.org/abs/2504.20073>  
  <https://arxiv.org/abs/2604.06268>

- **MAPPA**  
  用 process reward 给 multi-agent 系统中的每个 agent action 分配信用。这个思路可以迁移到安全场景：每次工具调用都可以评估是否越权、泄露或违反边界。  
  <https://arxiv.org/abs/2601.23228>

- **Agent Lightning**  
  把任意 Agent 执行过程形式化成 MDP transition，并用 hierarchical credit assignment 训练 Agent。适合把 SkillSafetyBench、AgentHarm、AgentDojo 这类 verifier 接成 reward provider。  
  <https://arxiv.org/abs/2508.03680>

安全场景中的 reward 可以拆成：

```text
terminal reward:
  task_success
  attack_success
  over_refusal

process reward:
  unsafe_tool_call
  scope_violation
  data_boundary_violation
  suspicious_skill_adherence
  prompt_injection_compliance
  missing_user_confirmation
  dangerous_external_action
```

这说明“稀疏安全奖励建模”是 Agent 安全对齐的核心问题之一，也是和 AgenticRL 最强的交叉点。

## 4. LLM 安全代码生成对齐

### 4.1 问题转变

安全代码生成不等于“代码能运行”。模型可能生成：

- 功能正确但有漏洞的代码。
- 看似安全但破坏功能的代码。
- 能绕过静态分析但仍可被利用的代码。
- 只在简单 toy benchmark 上安全，在真实仓库中失败的代码。

因此合理的评价目标是：

```text
correct-and-secure = passes functional tests AND avoids exploitable vulnerabilities
```

重要评测工作：

- **How Secure is Secure Code Generation?**  
  指出静态分析可能显著高估安全性，很多“安全输出”实际上不可运行或仍然存在风险。  
  <https://arxiv.org/abs/2601.07084>

- **RealSec-bench**  
  从真实 Java 仓库构造安全代码生成任务，同时评估功能正确性和安全性。  
  <https://arxiv.org/abs/2601.22706>

- **Detect--Repair--Verify**  
  研究检测、修复、验证闭环，用 runnable web app benchmark 评估漏洞修复。  
  <https://arxiv.org/abs/2603.23633>

## 5. 安全代码生成中的 RL / DPO / 偏好优化

### 5.1 经典基础线

- **CodeRL**  
  用执行反馈和强化学习提升代码生成，主要面向 functional correctness。虽然不是安全专用，但奠定了“代码生成 + 可执行反馈 + RL”的基础。  
  <https://arxiv.org/abs/2207.01780>

- **RLTF**  
  使用 unit test feedback 做代码生成强化学习，同样主要优化功能正确性，但方法可以迁移到安全 verifier。  
  <https://arxiv.org/abs/2307.04349>

### 5.2 安全代码对齐的新近工作

- **ProSec: Fortifying Code LLMs with Proactive Security Alignment**  
  使用 CWE 场景和偏好学习目标提升模型主动生成安全代码的能力。它的重点是 proactive security alignment，而不是只在用户显式要求安全时才修复。  
  <https://arxiv.org/abs/2411.12882>

- **Teaching an Old LLM Secure Coding**  
  使用 secure / insecure preference pairs 和 security reasoning，并提出 localized preference optimization。核心动机是：普通 DPO 对整段输出做偏好优化，而代码安全差异常常只集中在少数几行。  
  <https://aclanthology.org/2025.acl-long.1263/>

- **PrefGen**  
  在智能合约代码生成中使用 DPO 风格的偏好优化，同时考虑功能正确性、gas efficiency 和安全性。它体现了多目标偏好优化在代码安全中的价值。  
  <https://arxiv.org/abs/2506.03006>

- **SecCoderX**  
  使用 online RL 和 vulnerability reward model 来优化安全代码生成，同时尽量保持 utility。它关注“安全提升但功能下降”的 tradeoff。  
  <https://arxiv.org/abs/2602.07422>

- **SRCode / Vul2Safe**  
  研究 token-level rewards，用更细粒度的 reward 对安全相关 token 或 span 进行反馈。这个方向和代码安全很契合，因为漏洞常常由少数 token、API 调用或边界检查决定。  
  <https://arxiv.org/abs/2602.23407>

### 5.3 为什么 localized optimization 很重要

普通 DPO 把整段输出当成 preferred / rejected。对代码安全来说，这通常太粗：

```python
# 大部分程序可能都是对的。
# 漏洞可能只是一个缺失的 bounds check、
# 一个未转义 SQL 字符串、
# 一个不安全反序列化调用、
# 一个错误的 path join、
# 或一个不安全的 crypto mode。
```

因此更合理的是：

```text
whole-program reward:
  passes tests
  no exploit observed
  no high-confidence static finding

span-level reward:
  vulnerable line
  missing validation
  unsafe API
  insecure data flow

token / AST-level reward:
  dangerous function call
  missing sanitizer
  incorrect condition
  unsafe permission
  weak cryptographic primitive
```

可以结合程序分析做 localized preference：

```text
SAST trace / CodeQL path / Semgrep finding / AST diff / taint slice
  -> 定位 security-critical span
  -> 构造 secure vs insecure preference pair
  -> 只在相关 span 上施加 DPO / RL loss
```

这类方法比 whole-output DPO 更有信噪比，也更容易解释为什么模型变安全。

## 6. System-2 / CoT 能否用于安全代码对齐

你的直觉是对的：代码生成里显式 CoT 比数学推理少，原因包括：

- 最终产物通常应该是代码，而不是长篇推理。
- 长推理增加 token 成本。
- 推理内容可能污染输出格式。
- 代码任务已经有编译、测试、静态分析等外部反馈。
- 泛泛的安全 CoT 不一定能落到具体漏洞修复。

但这不代表 System-2 对齐不可行。更合理的做法是把安全推理作为训练时的过程监督，而不是推理时必须展示：

```text
训练时:
  prompt -> threat model / CWE reasoning / mitigation plan -> secure code

推理时:
  prompt -> secure code
```

或者让模型先进行隐藏的 security reasoning，再只输出最终代码。

相关工作：

- **SecPI**  
  训练模型内化 structured security reasoning，使其即使没有显式安全 prompt，也会考虑 CWE 和 mitigation。  
  <https://arxiv.org/abs/2604.03587>

- **MA-CoT**  
  使用 mitigation-aware reasoning，把 CWE mitigation guidance 和语言特定 safeguards 放入推理过程。它也说明 generic CoT 不一定可靠，甚至可能引入新漏洞。  
  <https://arxiv.org/abs/2605.24300>

关键点是：不是“推理越多越安全”，而是推理必须绑定到：

- 具体 CWE。
- 可利用路径。
- 数据流 / 控制流。
- 语言和框架相关防护。
- 最终代码中的实际 mitigation。

## 7. 可扩展成 AI 顶会论文的方向

### 7.1 Agent 方向：稀疏安全 reward 到 dense process reward

可以基于 SkillSafetyBench 这类 benchmark 做安全 RL：

```text
environment:
  task workspace + skills + verifier

terminal rewards:
  task_success
  attack_success
  over_refusal

process rewards:
  unsafe_tool_call
  suspicious_skill_obedience
  data_exfiltration
  unauthorized_persistence
  missing_confirmation
  external_action_risk

optimization:
  constrained RL
  Pareto-GRPO
  Pareto-DPO
  repair-trajectory self-evolution
```

可能贡献：

- 把稀疏最终 safety label 转换成 step-level credit。
- 学习 trajectory reward model，定位导致 unsafe behavior 的动作。
- 在不牺牲 task success 的情况下降低 attack success。

### 7.2 Agent 方向：trajectory-localized safe RL

不要只给整条轨迹一个 reward，而是定位安全失败发生在哪类动作：

```text
skill read:
  是否识别不可信指令？

tool call:
  是否越过用户授权范围？

file write:
  是否存在持久化 / 后门风险？

network/email action:
  是否发往未批准目的地？

memory update:
  是否污染长期记忆？
```

这可以把 FATE 的 failure repair 和 GiGPO / Agent Lightning 的 credit assignment 连接起来。

### 7.3 代码方向：process-localized preference optimization

一个可发表方案可以是：

```text
1. 生成多个候选代码。
2. 运行 unit tests、static analyzers、fuzzing、exploit tests。
3. 用程序分析定位 security-critical spans。
4. 构造 secure / insecure preference pairs。
5. 只在安全相关 span 上训练 localized DPO / RL。
6. 在 held-out CWE、held-out language、真实仓库上评估。
```

主张可以是：

```text
Localized, verifier-grounded preferences improve secure code generation
more reliably than whole-output DPO or generic security prompting.
```

中文表述就是：基于 verifier 和程序分析定位的局部偏好优化，比整段输出级 DPO 或泛泛安全提示更可靠。

### 7.4 代码方向：Agentic secure coding alignment

把安全代码生成看成 Agent 任务，而不是 one-shot generation：

```text
write code -> run tests -> run SAST -> inspect trace -> patch -> run exploit test
```

训练目标不只是最终代码安全，还包括验证行为是否高效、是否正确理解 analyzer trace、是否避免 reward hacking：

```text
reward = functional_correctness
       + security_pass
       + efficient_verification
       - analyzer_evasion
       - functionality_removal
```

这个方向能把安全代码生成、Agent 安全对齐和 AgenticRL 连起来。

### 7.5 System-2 安全代码对齐

可以训练模型产生私有或隐藏的安全推理：

```text
teacher trace:
  identify CWE -> reason about exploit path -> choose mitigation -> produce code

student objective:
  secure final code + optional process reward for correct latent reasoning
```

适合的漏洞类型：

- SQL injection。
- Command injection。
- Path traversal。
- Deserialization。
- Authentication / authorization bypass。
- Insecure cryptography。
- Memory safety。
- XSS / template injection。

关键实验应该比较：

- 无 CoT。
- generic CoT。
- CWE-aware CoT。
- mitigation-aware CoT。
- 隐式 System-2 training。

并验证最终代码是否真的更安全，而不是只是解释更像安全专家。

## 8. Reward hacking 风险

无论 Agent 安全还是安全代码生成，都要警惕 reward hacking。

Agent 可能学会：

- 不产生日志，让 verifier 找不到证据。
- 避免输出被检测 artifact。
- 为了安全 reward 过度拒绝正常任务。
- 通过修改环境隐藏失败。

代码模型可能学会：

- 满足静态分析器但不修复真实 exploit。
- 删除功能来避免漏洞路径。
- 生成不可运行但“看起来安全”的代码。
- 只适配训练时见过的 CWE template。

因此强实验需要包括：

```text
static-only reward vs dynamic exploit reward
seen CWE vs held-out CWE
toy tasks vs real repositories
single-step generation vs multi-turn agent coding
normal prompts vs prompt-injected skills/workspaces
functionality-preserving security repair
```

## 9. 总结

最有潜力的研究方向不是简单“把 RL / DPO 用到安全上”，而是：

```text
用 verifier-grounded、localized、trajectory-aware 的 reward modeling，
在稀疏、延迟、多目标的安全反馈下对齐 Agent 和代码模型。
```

对 Agent 来说，对齐单位应该是完整 action trajectory。  
对安全代码生成来说，对齐单位往往应该是 security-critical code span。  
两者共同的核心技术难点是：如何在稀疏安全反馈下做可靠 credit assignment，同时不牺牲正常任务能力。
