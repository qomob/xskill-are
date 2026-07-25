# 维度 7：运行时正确性 `runtime`（可选）

> 本维度为**可选维度**，仅当外部提供 `runtimeTrace`（符合 [xskill-trace-v1](runtime-trace-schema.md)）时启用。
>
> **核心约束：** xskill-are 消费 trace，不执行 Skill。D7 评判的是"trace 中呈现的运行时正确性"，而非 xskill-are 自身的运行结果。
>
> **不参与多法官共识：** trace 是事实数据，不需要法官透镜共识。R1-R4 由程序化判定 + 1 次 LLM Judge（仅 R3）组成。
>
> **不参与安全一票否决：** D7 分数不影响 safety 一票否决逻辑。但 `severity: fatal` 的运行时缺陷会进入 `fatalFlaws`，作为阻断信号。

---

## 启用条件

| 条件 | D7 状态 |
|------|---------|
| 输入未提供 `runtimeTrace` | `NOT_ASSESSED`（不进入综合分，输出报告标注） |
| `runtimeTrace.schemaVersion !== "xskill-trace-v1"` | `NOT_ASSESSED`（格式不匹配，降级） |
| `runtimeTrace.cases` 为空数组 | 40 分（保守分，标注"运行时证据不足"） |
| `runtimeTrace.cases` 至少 1 条 | 启用 R1-R4 判定 |

---

## 4 条硬断言（R1-R4）

### R1：工具调用成功率

**判定：** 统计所有 `cases[].toolCalls[]`，计算 `success === true` 的比例。

| 通过阈值 | 分值贡献 |
|---------|---------|
| ≥ 80% | R1 PASS |
| 60%-79% | R1 PARTIAL（贡献 0.5） |
| < 60% | R1 FAIL |

**归一化规则：**
- 单 case 内无工具调用（`toolCalls.length === 0`）时，该 case 不计入 R1 分母（既不惩罚也不奖励）
- 所有 case 都无工具调用时，R1 退化为 PASS（无调用即无失败）

### R2：无 fatal 错误

**判定：** 遍历所有 `cases[].error`，检查是否存在 `type === 'fatal'`。

| 条件 | 结果 |
|------|------|
| 无任何 `error` 字段 | R2 PASS |
| 存在 `error` 但全部 `type !== 'fatal'` | R2 PASS |
| 存在任一 `error.type === 'fatal'` | R2 FAIL + 写入 `fatalFlaws` |

**fatal 范例：** 进程崩溃（`exitCode !== 0` 且 error.type=fatal）、核心工具完全不可用、断言失败导致中断。

### R3：多轮上下文保持

**判定：** 对 `cases[].turns.length > 1` 的 case，由 LLM Judge 判断"最后一轮输出与第一轮输入的语义相关性"。

**Judge prompt 模板：**

```
你是一个严格的语义相关性评判者。

输入：
- 第一轮用户输入：{turns[0].input}
- 最后一轮 Agent 输出：{turns[last].output}

任务：判断"最后一轮输出"是否仍在回答"第一轮输入"的核心问题。
- true：输出与输入语义相关，未跑题、未遗忘上下文
- false：输出与输入无关，或明显遗忘前文上下文

仅返回 JSON：{"relevant": true|false}
```

**统计：**

```
relevantCases = Σ(LLM Judge 返回 true 的 case 数)
multiTurnCases = Σ(turns.length > 1 的 case 数)
rate = relevantCases / multiTurnCases
```

| 通过阈值 | 结果 |
|---------|------|
| rate ≥ 80% | R3 PASS |
| rate < 80% | R3 FAIL |

**降级：** 当所有 case 都是单轮（`multiTurnCases === 0`）时，R3 退化为 PASS（无多轮即无上下文丢失风险），并在报告中标注"未评测多轮场景"。

### R4：artifact 有效性

**判定：** 遍历所有 `cases[].artifacts[]`，检查声明产出的文件是否全部 `exists === true && sizeBytes > 0`。

| 条件 | 结果 |
|------|------|
| 无 artifact 声明 | R4 PASS（无声明即无失败） |
| 全部 `exists && sizeBytes > 0` | R4 PASS |
| 存在 `exists === false` 或 `sizeBytes === 0` | R4 FAIL |

---

## 综合分计算

```
passedCount = (R1 PASS?1:0) + (R2 PASS?1:0) + (R3 PASS?1:0) + (R4 PASS?1:0)
partialCount = (R1 PARTIAL?0.5:0)

runtime = 分级映射(passedCount + partialCount)
```

| passedCount + partialCount | runtime 分数 |
|:---:|:---:|
| 4.0 | 90 |
| 3.5 | 80 |
| 3.0 | 70 |
| 2.5 | 60 |
| 2.0 | 50 |
| ≤ 1.5 | 30 |

**设计理由：** 4/4 对应 90 分而非 100，保留"运行时正确性满分不等于完美"的诚实空间（trace 只是抽样，不能证明所有场景都正确）。

---

## runtimeIssues 提取规则

从 trace 中提取的运行时问题写入 `AIReport.runtimeIssues`：

| 来源 | type | severity | 提取逻辑 |
|------|------|---------|---------|
| `error.type === 'fatal'` | `crash` | fatal | 全部提取 |
| `error.type === 'major'` | `tool_failure` | major | 全部提取 |
| R1 FAIL（成功率 < 60%） | `tool_call_error` | major | 汇总为 1 条 |
| R3 FAIL（上下文丢失） | `multi_turn_loss` | major | 汇总为 1 条 |
| R4 FAIL（artifact 缺失） | `artifact_invalid` | major | 每个缺失文件 1 条 |
| `error.type === 'minor'` | `minor_issue` | minor | 可选提取（默认不提取） |

**fatal severity 的 issue 同时进入 `fatalFlaws`**，作为阻断信号（与 D3 鲁棒性失败、D4 安全失败并列）。

---

## 边界与诚实声明

1. **trace 是抽样证据，不是穷尽证明** — D7 高分不代表 Skill 在所有场景都正确，只代表"被测 case 都过了"
2. **case 覆盖度依赖外部工具** — xskill-are 不控制 case 数量与质量，调用方需自行保证覆盖度
3. **R3 LLM Judge 有主观性** — "语义相关性"判断存在边界模糊（如创造性跑题 vs 真正遗忘），未启用多法官（成本与价值不匹配）
4. **不验证业务正确性** — D7 只看"调用层"和"结构层"正确性，业务语义正确性由 D1 评判

> **建议：** D7 应与 D1-D6 联合解读。D1-D6 高分但 D7 低分的 Skill，通常是"写得漂亮但跑不起来"的典型；D7 高分但 D1 低分的 Skill，则是"能跑但产出价值低"。
