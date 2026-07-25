# XSkill Runtime Trace Schema — xskill-trace-v1

> 本文件定义 xskill-are D7「运行时正确性」维度的可选输入契约。
>
> **设计原则：** xskill-are 消费 trace，不执行 Skill（守住 SkillForge Core Constraint #1）。trace 由外部运行时工具（如 skill-up / Claude Code evals / 自建 agent 测试框架）产生，xskill-are 只做"trace 解读"。
>
> **版权声明：** 本 schema 为 xskill-are 原创设计，字段命名与结构不引用任何第三方 schema。trace 内容为"事实数据"（非作品），消费行为零版权风险。

---

## Schema 版本

```
schemaVersion: "xskill-trace-v1"
```

xskill-are 仅消费 `xskill-trace-v1` 格式。其他格式（含第三方 schema）需由调用方转换为本格式后再传入。`schemaVersion` 不匹配时，D7 降级为 `NOT_ASSESSED`（见 [degradation-matrix.md](degradation-matrix.md)）。

---

## 顶层接口

```typescript
interface XSkillRuntimeTrace {
  /** 固定值 "xskill-trace-v1" */
  schemaVersion: 'xskill-trace-v1';
  /** 产生该 trace 的 Agent 引擎标识 */
  engine: string;
  /** 引擎使用的模型标识 */
  model: string;
  /** 测试用例 trace 列表（至少 1 条） */
  cases: TraceCase[];
  /** 可选：无 Skill 基线输出（用于 D1 A6 对照增益断言） */
  baselineOutputs?: BaselineEntry[];
  /** trace 生成时间 ISO 8601 */
  generatedAt: string;
}

interface BaselineEntry {
  /** 关联的 caseId（与 cases[].caseId 对应） */
  caseId: string;
  /** 无 Skill 时同一 prompt 的输出文本 */
  output: string;
}
```

---

## Case 与 Turn

```typescript
interface TraceCase {
  /** 用例标识（与外部工具的 case id 对应，由调用方填入） */
  caseId: string;
  /** 多轮对话记录（至少 1 轮） */
  turns: Turn[];
  /** 工具调用记录（含 MCP 调用） */
  toolCalls: ToolCall[];
  /** 声明产出的 artifact 文件 */
  artifacts: Artifact[];
  /** 最终输出文本 */
  finalOutput: string;
  /** 进程退出码：0=正常，非 0=异常 */
  exitCode: number;
  /** 执行耗时（毫秒） */
  durationMs: number;
  /** 可选：错误信息（fatal 错误会进入 fatalFlaws） */
  error?: TraceError;
}

interface Turn {
  /** 轮次序号（从 1 开始） */
  index: number;
  /** 该轮输入（用户或系统） */
  input: string;
  /** 该轮 Agent 输出 */
  output: string;
}

interface ToolCall {
  /** 工具名（如 read_file / search_web / mcp__github__create_issue） */
  name: string;
  /** 调用是否成功：true=返回有效结果，false=报错/超时/拒绝 */
  success: boolean;
  /** 可选：失败原因 */
  errorMessage?: string;
  /** 可选：调用耗时（毫秒） */
  durationMs?: number;
}

interface Artifact {
  /** 声明产出的文件相对路径 */
  path: string;
  /** 文件是否存在且非空 */
  exists: boolean;
  /** 文件大小（字节）；不存在时为 0 */
  sizeBytes: number;
}

interface TraceError {
  /** 错误类型分级 */
  type: 'fatal' | 'major' | 'minor';
  /** 错误消息 */
  message: string;
  /** 可选：发生位置（caseId / turn index / toolCall name） */
  location?: string;
}
```

---

## 字段语义说明

### `error.type` 分级

| 类型 | 含义 | D7 处理 |
|------|------|---------|
| `fatal` | 进程崩溃 / 断言失败 / 工具完全不可用 | R2 断言失败；写入 `runtimeIssues` 并进入 `fatalFlaws` |
| `major` | 部分工具失败 / 超时 / 输出截断 | R1 工具成功率计算扣分 |
| `minor` | 延迟高 / 警告 | 不影响 R1-R4 判定，仅记录 |

### `toolCalls.success` 判定

- `true`：工具被调用且返回有效结果（无论业务语义是否正确，D7 只看"调用层成功"）
- `false`：工具调用抛错 / 超时 / 被引擎拒绝 / 返回空结果

业务语义正确性由 D1 业务增益度评判，D7 只评"调用层正确性"。

### `baselineOutputs`（可选）

仅当外部工具已做过 "with skill vs without skill" 对照实验时填入。xskill-are 在 D1 业务增益度中触发 A6 对照增益断言（详见 [rubric-business.md](rubric-business.md)）。未提供时 A6 跳过，D1 退化为 A1-A5 五条断言。

---

## 最小合法 trace 示例

```json
{
  "schemaVersion": "xskill-trace-v1",
  "engine": "claude-code",
  "model": "claude-sonnet-4-6",
  "generatedAt": "2026-07-25T10:00:00Z",
  "cases": [
    {
      "caseId": "hello-world",
      "turns": [
        {
          "index": 1,
          "input": "请生成 Hello World 程序",
          "output": "我来生成一个 Hello World 程序..."
        }
      ],
      "toolCalls": [
        { "name": "write_file", "success": true, "durationMs": 120 }
      ],
      "artifacts": [
        { "path": "hello.py", "exists": true, "sizeBytes": 45 }
      ],
      "finalOutput": "已生成 hello.py",
      "exitCode": 0,
      "durationMs": 3200
    }
  ]
}
```

---

## 适配第三方工具的转换指引

xskill-are 不内置任何第三方 schema 转换器。调用方负责将外部 trace 转换为 `xskill-trace-v1`：

| 外部工具字段（示例） | xskill-trace-v1 对应字段 |
|---------------------|------------------------|
| 外部 case 标识 | `cases[].caseId` |
| 外部会话消息序列 | `cases[].turns[]` |
| 外部工具调用记录 | `cases[].toolCalls[]`（需归一化 `success` 布尔） |
| 外部产物文件清单 | `cases[].artifacts[]`（需补全 `exists` / `sizeBytes`） |
| 外部错误对象 | `cases[].error`（需映射到 `fatal/major/minor`） |

> 转换逻辑由调用方实现，xskill-are 不对第三方格式做任何承诺。
