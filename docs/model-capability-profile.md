# 大模型能力画像子系统 — 通用设计文档

> 本文档描述一个**与任何具体 Agent 产品、任何具体大模型供应商解耦**的能力画像子系统。  
> 它适用于「底层大模型可在运行时切换」的任意 Agent 框架。接入方（某 Agent 运行时、某供应商、  
> 某托管服务）只是消费者/实现者，不进入本规格的强制内容。  
> 文末 §14 是**非规范性**的「设计灵感」注脚，不构成约束。



---

## 1. 问题陈述

一个 Agent 在运行时可在多个底层大模型之间切换（不同供应商、不同版本、甚至同一名称换后端）。  
我们希望为每个底层模型维护一份「能力画像」，并让它产生价值：

- **单 Agent 场景**：人和 Agent 都能读画像，据此调整 / 补充指令，把任务执行得更好（自校准）。
- **多 Agent 场景**：Agent 读各模型画像，自行判断把子任务路由给哪个模型（路由选型）。
- **人读场景**：画像是一份人类可读的「各模型擅长什么 / 不擅长什么」记录。

难点在于：被画像的对象（底层模型）是**可变**的。这击穿了「固定编队里每只智能体观察同伴」的隐含假设，  
因此来源合成、证据池、蒸馏审批、真相源、应用、演化每一环都必须用平台无关的方法推导。

---

## 2. 设计不变量（六道门）

任何合规实现都必须守住以下六条。缺任意一条，画像都会退化为「写死的 config」或「自嗨的简历」：

1. **可追加的证据**：所有原始观察进入一个只追加的「信号池」，每条带 `evidence`，不可静默改写历史。
2. **可重读的真相源**：权威画像是一份可被运行时重新读取的文本文件，修改后即生效，无需重启进程。
3. **人工审批闸门**：任何「结论层」写入（把信号收敛成画像）必须经人类策展者（Curator）确认；算法只收集证据、不给结论。
4. **偏好闭环捕获**：关于「使用者品味」的结论必须走 `提议 → 人批 → 可靠落盘` 的闭环，且**捕获侧必须有低摩擦机制**（命令/钩子/主动提示），否则闭环会静默死亡（有目录无写入 = 零增长）。taste 类结论天然只归 Curator。
5. **证据可版本化可复核**：能力事实不能只来自人类观察——人类未必察觉模型的隐性短板。证据须来自**可版本化、可复现**的信号源（自动 eval / 结构化评测 / 轨迹回放），并带 `judge` 版本与复现信息，否则声明不可信。
6. **演化可回滚**：画像的每一次变更都由人类持有「价值 / 冻结 / 批准」权，且**任何变更可回滚**。系统不得自动把信号收敛成结论并落盘。

---

## 3. 角色（与平台无关）

| 角色                         | 职责                                                                                      | 是否落笔权威文件 |
| -------------------------- | --------------------------------------------------------------------------------------- | -------- |
| **Observer（观察者）**          | 在运行中产生「提议信号」：带 `modelName` + `source` + `evidence`。可由当前模型自反思、同伴模型、harness 轨迹、eval 指标担任。 | **否**    |
| **Curator（策展者）**           | 人类使用者。审阅信号池，把信号收敛为画像并落笔权威文件；是唯一权威作者；持有冻结/回滚权。                                           | **是**    |
| **Store（真相源）**             | 按 `modelName` 索引的每模型文件集合；提供读 / 写（仅由 Curator 调用）。                                        | 托管       |
| **Router / Consumer（消费者）** | 运行时读画像：单 Agent 做自校准，多 Agent 做路由选型；只读，不写。                                                | 否        |

**关键约束**：Observer 与 Curator 必须分离——**被画像的模型绝不能直接落笔自己的权威画像**  
（主体 ≠ 落笔人），否则弱模型会写出 flattering 的自画像。taste 类结论天然只归 Curator。

---

## 4. 数据模型：每模型一份文件

真相源是一个目录，其中每个底层模型对应一个文件，文件名为模型**名称**（稳定），例如：

```
profiles/
  claude-opus-5.md
  claude-fable-5-1.md
  gpt-6-astra.md
  gpt-5-6-sol.md
```

> 模型**名称**稳定，但底层 **id / 来源供应商**可能变化（同名称换后端）。  
> 处理方式是文件内加 `source` / `backendId` 字段，**不新建文件**。

### 单文件格式

```markdown
# model-profile: <modelName>

source: <backendId / provider，可随来源变化>
provenance:
  version: 1
  updated: 2026-09-08
  primarySources: [operator, self-reflection, peer, eval]

# —— 结构化表头：供 Router 机器解析（路由用）——
l0RosterSummary: <≤50字，速查>
routingSignals:
  peakCapabilities: [擅长A, 擅长B]
  antiSignals: [不擅长X, 不擅长Y]
  tripwire: <翻车熔断信号，命中即降级/改派>

# —— 叙事体：供自校准与人读（品味/边界/证据）——
profile:
  nativeStrengths: <原生峰值能力>
  underrated: <被低估的能力>
  badInstincts: <坏直觉——必填，防只写优点>
  complements: <与谁/什么互补，反模式>
  tasteNotes: <贴合本使用者品味的观察，归 Curator>

# —— 证据：每条结论须能在此回溯 ——
evidence:
  - id: ev-001
    source: <operator|self|peer|eval>
    judge: <证据来源标识 + 版本，如 eval-suite@v3；eval 类必填>
    reproducible: <复现方式 / 数据集，可选>
    observation: <事实>
    snippet: <可引用片段>
```

**字段硬约束**：`badInstincts` 必填；`tripwire` 必填；任意 `profile.*` 结论必须能在 `evidence` 中找到对应项，  
否则该结论不得在画像中落笔（fail-closed）。`eval` 类证据必须带 `judge` 版本，否则不可作为能力声明依据。

### 模型上线性 probe（onboarding probe）

当某 `modelName` 第一次接入、或同一名称更换 `source` 后端时，不应**假设既有画像通用**。  
实现方应在落笔前做一次**实证能力 probe**，产出该模型相对通用模板的能力差距矩阵，写入 `evidence` 与 `profile`，  
避免「同名换源后能力断崖」被静默忽略。

---

## 5. 信号池（Observer 的产出）

信号池是一份只追加的文件（或等价的后端），结构：

```markdown
## [<date>] <type> · model=<modelName> · source=<operator|self|peer|eval>
proposed: <一句事实提议>
evidence: <可引用片段>
judge: <eval 类必填：来源+版本>
status: proposed   # proposed → (Curator 采纳/驳回)
```

- `type` ∈ {strength, weakness, quirk, tripwire, context-fit, taste}
- Observer **只写信号池**，永不直写 `profiles/<modelName>.md`。
- 信号池是提议层；权威文件是结论层。两层分离是治理的核心。

---

## 6. 生命周期

```
 Observer ──提议──▶ 信号池(append-only)
                        │
                        ▼
                  Curator 审阅（人）
                        │ 采纳/收敛（价值/冻结/批准权在 Curator）
                        ▼
              profiles/<modelName>.md（权威，version+1，可回滚）
                        │
            ┌───────────┴────────────┐
            ▼                        ▼
   单 Agent：自校准指令        多 Agent：Router 选型
            │                        │
            └────── 新信号回流 ────────┘
```

1. **capture**：Observer 在任务中或任务后产生信号，写入信号池（带 `modelName`）。taste 类信号须有低摩擦捕获入口。
2. **curate**：Curator 周期/按需审阅信号池，把信号收敛为画像 diff，确认后落笔权威文件，`provenance.version +1`。  
   创建者不自批（职责分离）。每条结论必须能在池中找到 `evidence`（eval 类带 `judge` 版本）。
3. **apply（单 Agent）**：当前模型运行任务前/中读自身画像的 `badInstincts` / `tripwire` 做自校准，并据 `tasteNotes` 调整指令风格。
4. **apply（多 Agent）**：Router 读各模型 `routingSignals` 选择子任务承接模型；命中某模型 `tripwire` 即降级或改派。
5. **rollback**：任意历史版本可由 Curator 回滚；Store 须保留版本历史。
6. **feedback**：应用产生的新表现回流为信号池的新提议，闭环。

---

## 7. 接口契约（伪代码，平台无关）

```ts
// —— Observer：只产提议，不写权威文件 ——
interface Observer {
  emit(signal: {
    modelName: string;
    type: "strength" | "weakness" | "quirk" | "tripwire" | "context-fit" | "taste";
    source: "operator" | "self" | "peer" | "eval";
    proposed: string;
    evidence: string;                 // 可引用片段，必填
    judge?: string;                   // eval 类必填：来源+版本
  }): void;                           // 仅追加到信号池
}

// —— Curator：唯一权威作者，人 ——
interface Curator {
  review(pool: SignalPool): ProposedDiff;          // 人类审阅信号池
  approve(diff: ProposedDiff, modelName: string): void;  // 落笔 profiles/<modelName>.md, version+1
  rollback(modelName: string, toVersion: number): void; // 回滚
  freeze(modelName: string): void;                 // 冻结（暂停自动/外部修改）
  // 约束：diff 中每条结论必须能在 pool 找到 evidence（eval 类带 judge），否则拒绝
}

// —— Store：按 modelName 索引的真相源 ——
interface Store {
  read(modelName: string): ModelProfile;           // 运行时重读，无需重启
  write(modelName: string, profile: ModelProfile): void;  // 仅 Curator 调用
  history(modelName: string): ModelProfile[];      // 版本历史，供回滚
  list(): string[];                                // 所有已知 modelName
}

// —— Router / Consumer：只读 ——
interface Router {
  select(task: Task, ctx?: RoutingContext): string;   // 读各 profile.routingSignals 返回 modelName
  selfCalibrate(modelName: string): CalibrationHint;  // 读 badInstincts/tripwire/tasteNotes
}
```

**框架必须提供**：一个 `Store`（目录或等价后端，位置可配置）、一个 `Observer` 接入点、  
以及「Curator = 人类」这一治理约定。其余角色可由框架自行实现。

---

## 8. 多 Agent 路由用法

- Router 在派发子任务前调用 `select(task, ctx)`，读取候选模型的 `peakCapabilities` / `antiSignals` / `tripwire`，返回最合适的 `modelName`。
- **实时路由上下文**：`ctx` 可携带任务当下的约束（上下文长度、延迟预算、成本上限、保密边界）。画像提供模型能力，上下文提供即时约束，二者正交——路由是「能力 × 约束」的交集求解，不是单纯能力排序。
- 同伴模型可作为 Observer 为彼此产提议（跨模型 review），但落笔仍归 Curator——保留「主体≠落笔人」。
- 路由只消费**结构化表头**，不解析叙事体，保证机器可读、低延迟。

## 9. 单 Agent 自校准用法

- 当前模型运行前读自身画像：`badInstincts` 提醒自己别踩坑，`tasteNotes` 调整表达风格，`tripwire` 设熔断。
- 人和 Agent 都能据画像**补充指令**（如「这模型长上下文强但易过度展开，指令里要求简洁」）。
- **越用越贴合使用者品味**：taste 信号经闭环捕获（§2 不变量4）固化进 `tasteNotes`——每次 Curator 策展都把使用者偏好写入，且捕获侧须有低摩擦入口，否则闭环静默死亡。

---

## 10. 模型身份处理

- **键 = 模型名称**（如 `claude-opus-5`），稳定可版本化，画像可长期累积。
- **id / 供应商可变**：同名换后端时，更新文件内 `source` / `backendId`，不新建文件、不清空历史。
- **换源须 probe**：同一名称换 `source` 后，按 §4「上线性 probe」实证差距，不假设模板画像通用。
- 若同一名称能力因来源差异显著分化，Curator 可在文件内用 `source` 分段记录，而非分裂成多文件。

---

## 11. 可移植性清单（接入方自检）

- [ ] 真相源目录位置可配置，不写死平台路径。
- [ ] Observer 只写信号池，不触达权威文件。
- [ ] 权威文件每次访问重读（内容哈希缓存可选），修改无需重启。
- [ ] 结论层写入必经人类 Curator，且每条结论可回溯 evidence（eval 带 judge 版本）。
- [ ] 画像按 `modelName` 索引，支持同名换来源 + 上线性 probe。
- [ ] Curator 持有冻结/回滚权，任何变更可回滚。
- [ ] taste 捕获有低摩擦入口（命令/钩子/提示）。
- [ ] 文档/代码不含对任何具体 Agent 产品或供应商的硬编码依赖。

---

## 12. 反模式（不要做）

- ❌ 让被画像的模型直接写自己的权威画像（自夸偏差，结构性）。
- ❌ 把画像塞进数据库而失去「可重读文本 + 可 diff + 可人工校订」的特性。
- ❌ 算法自动把信号收敛成结论（绕过 Curator 闸门）。
- ❌ 画像只写优点，缺 `badInstincts` / `tripwire`。
- ❌ 把「画像」焊死在某一产品的文件布局 / 内存结构里（失去通用性）。
- ❌ 假设模板画像跨后端通用，同名换源后不 probe（能力断崖被静默忽略）。
- ❌ 只有品味目录、没有低摩擦捕获机制（闭环静默死亡，零增长）。
- ❌ 能力声明来自不可版本化、不可复现的证据（数字不可信）。

---

## 13. 开放问题（未来可扩展，非当前强制）

- **置信度/状态**：个人场景用 `proposed/curated` 二分即可；多租户生产路由可加 `confidence` 与自动衰减，但须保留 Curator 终局闸门。
- **在线 bandit**：高频路由可并行维护统计胜率矩阵，与叙事画像互为补充，不构成替代。
- **能力矩阵并存**：纯选型可额外维护结构化 capability vector，本规格的表头已为其预留接口。
- **实时路由上下文来源**：`ctx` 的约束由谁提供（harness / 任务元数据 / 用户），由各框架自行决定。

---

## 14. 设计灵感（非规范性注脚）

本设计在推导过程中参考了多智能体系统中「模型能力画像 / 认知路由 / 评估证据回流」的一类既有实践，  
作为问题空间与失败模式的灵感来源。**本节不构成规范约束**，也不要求任何接入方复现其具体实现。  
通用不变量（§2 六道门、§4 probe、§9 品味闭环）是独立推导的结果，可独立成立。
