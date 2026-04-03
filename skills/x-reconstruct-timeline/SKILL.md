---
name: x-reconstruct-timeline
description: 
  Use this skill to reconstruct what actually happened during execution from XCodeMap runtime data.
  Apply it when the user describes a sequence of actions, reproduces a bug, or when execution is uncertain (e.g., behavior differs, something may not have run)—even if they don’t explicitly ask for tracing.
  Maps user operations to backend calls, threads, and call chains based on observed data only.
---

# 1. When to use

- 用户描述一系列**操作步骤**，需要还原执行过程
- 复现 BUG，需要梳理：
  - 用户操作 → 实际执行链路
- 需要确认某次操作触发了哪些：
  - 后端接口
  - 线程
  - callId
- 需要分析**多次操作之间的执行差异**
- 存在 execution uncertainty（如：
  - “为什么没生效”
  - “是不是没执行”
  - “两次行为不一致”）

---

# 2. Rules

## Ground Truth

- Runtime data 是**完整录制的真实执行**
- 必须视为：
  > 唯一可信的 execution ground truth

- 不得假设：
  - 用户操作没有发生
  - bug 没被录到
  - 数据缺失关键执行

---

## Evidence Interpretation

- In scope + hit → **Strong Positive**
- In scope + no hit → **Strong Negative**
- Out of scope / no data → **Unknown**

---

## Reconstruction Constraints（核心）

- 只允许基于**已观测数据**重建执行过程
- 不得：
  - 推理未发生的执行
  - 补全缺失链路
  - 猜测隐含调用

- reconstruction 必须是：
  > 用户操作时间线 ↔ 执行时间线 的映射

---

## Prohibited

- ❌ 不解释 why（原因）
- ❌ 不构建因果链（causal chain）
- ❌ 不做 root cause analysis
- ❌ 不基于代码或经验补全执行

---

# 3. Workflow

1. 解析用户描述的操作流程（按时间顺序拆解）
2. 读取 `XCODEMAP_RUNTIMEDATA_API.md`
3. 调用 `checkRecordDataAndScope`
  - 确认是否存在 runtime 数据
  - 明确录制范围（scope）
4. 对每个操作：
  - 选择最小必要 observation points
  - 优先使用：
    - `findFunctionCalls`
    - `observeCallNode`
    - `observeSourceExecution`
5. 构建映射：
  - 用户操作 → 后端入口 → 调用链 → 分支结果
6. 若存在多次操作：
  - 明确对比执行差异（哪些步骤未发生 / 分支不同）
7. 输出结构化 reconstruction 结果（仅事实）

---
# 4. Output format

必须按步骤输出 execution reconstruction，每一步严格遵循以下结构：

---

## <step number>

**用户操作：**  
<描述用户行为>

**后端对应的入口：**

- **线程：** `<thread name>`（threadId: <id>）

- **调用链：**
  - <Class.method>（callId: **c_xxx**）
  - …

- **关键逻辑：**
  - 关键参数（如 currCallId / nextCallId）
  - 分支条件与路径
  - 返回结果 / 下一个节点

---

## 强制要求（每一步必须满足）

- ✔ 用户操作与执行必须**一一对应**
- ✔ 必须包含线程信息（name + threadId）
- ✔ 必须包含关键 callId
- ✔ 调用链必须最小但完整（不得冗余或缺失关键节点）
- ✔ 必须包含关键参数 / 分支 / 返回结果（如存在）

---

## 多次操作（差异分析）

若用户描述包含多次操作（如重复点击、不同结果）：

- 必须逐步对齐对比
- 明确指出：
  - 哪一步执行发生差异
  - 哪个调用未出现（Strong Negative）
  - 或分支路径不同

---

## 数据不足

若无法完成 reconstruction：

> **insufficient reconstruction**

不得：

- 推测执行过程
- 用代码逻辑替代 runtime 证据