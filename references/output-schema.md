# 输出数据契约 — TypeScript 接口定义

> 评测完成后输出 AIScore + AIReport 两个结构化对象。严格遵循此 schema。

---

## AIScore 接口

```typescript
interface AIScore {
  /** 综合评分 0-100（惩罚后，安全一票否决后） */
  overall: number;
  /** 业务增益度 0-100 */
  business: number;
  /** 提示词工程 0-100 */
  prompt: number;
  /** 混沌鲁棒性 0-100 */
  robustness: number;
  /** 安全合规 0-100 */
  safety: number;
  /** 生态兼容性 0-100 */
  composability: number;
  /** 性价比 0-100 */
  cost: number;
  /** 运行时正确性 0-100，或 null（D7 未评测）。详见 rubric-runtime.md */
  runtime?: number | null;
  /** 运行时证据来源：'external_trace'（消费 runtimeTrace）| 'native_probe'（保留）| null（未评测） */
  runtimeSource?: string | null;
  /** 等级：卓越|优秀|良好|合格|不合格 */
  grade: string;
  /** HRR 分级：S|A|B|C */
  hrrTier: string;
  /** 评测时间 ISO 8601 */
  evaluatedAt: string;
  /** 评测模型版本 */
  modelVersion: string;
  /** 评测器版本（xskill-are 版本，用于追踪评分漂移） */
  evaluatorVersion: string;
  /** 校准漂移量（锚点实测D1 - 预期D1 的绝对值） */
  calibrationDelta: number;
  /** 校准状态：CALIBRATED | DRIFT_WARNING | DRIFT_ALERT | NOT_ASSESSED */
  calibrationStatus: string;
  /** 多法官一致性结果 */
  judgeConsensus: JudgeConsensus;
}

interface JudgeConsensus {
  /** 法官一致比例（全一致断言数/总断言数） */
  agreement: number;
  /** 分歧断言列表 */
  disputedAssertions: string[];
  /** 是否触发仲裁 */
  arbitrationTriggered: boolean;
}
```

### 等级判定

```typescript
function getGrade(overall: number): string {
  if (overall >= 90) return '卓越';
  if (overall >= 80) return '优秀';
  if (overall >= 70) return '良好';
  if (overall >= 60) return '合格';
  return '不合格';
}
```

### HRR Tier 判定

```typescript
function getHrrTier(overall: number): string {
  if (overall >= 85) return 'S';
  if (overall >= 68) return 'A';
  if (overall >= 48) return 'B';
  return 'C';
}
```

---

## AIReport 接口

```typescript
interface AIReport {
  /** 优势（3-5 条） */
  strengths: string[];
  /** 不足（2-3 条） */
  weaknesses: string[];
  /** 最适用场景 */
  bestFor: string[];
  /** 不适用场景 */
  notFor: string[];
  /** 致命缺陷（红队/安全/运行时 fatal 未通过项） */
  fatalFlaws: string[];
  /** HRR 分级：S|A|B|C */
  hrrTier: string;
  /** 测试用例结果 */
  testCases: TestCase[];
  /** 元反思结果 */
  metaReflection: MetaReflection;
  /** 运行时发现的缺陷（仅当 runtimeTrace 提供时；未提供时字段缺失或为空数组） */
  runtimeIssues?: RuntimeIssue[];
}

interface RuntimeIssue {
  /** 问题类型：crash | tool_call_error | mcp_failure | multi_turn_loss | artifact_invalid | timeout | minor_issue */
  type: string;
  /** 严重度：fatal | major | minor（fatal 会同时进入 fatalFlaws） */
  severity: string;
  /** 发生的 case/turn/tool 定位（如 "caseId:hello-world / turn:2 / tool:write_file"） */
  location: string;
  /** 证据片段（trace 摘录或错误消息） */
  evidence: string;
}

interface MetaReflection {
  /** 是否通过（flaggedIssues < 2 为 pass） */
  passed: boolean;
  /** 标记的问题数量 */
  flaggedIssues: number;
  /** 标记的维度（问题定义/假设/推理/证据/替代解释/边界条件/目标/不确定性） */
  flaggedDimensions: string[];
  /** 触发回退的维度（如有） */
  rolledBackDimensions: string[];
}

interface TestCase {
  /** 测试用例名 */
  name: string;
  /** 输入（截断 200 字） */
  input: string;
  /** 是否通过 */
  passed: boolean;
  /** 备注 */
  note: string;
}
```

---

## 惩罚函数

```typescript
// v2.1 线性拉伸：以 70 为支点，向上向下各放大 1.5 倍
function applyPenalty(rawScore: number): number {
  return Math.max(0, Math.min(100, Math.round(rawScore * 1.5 - 35)));
}
```

| 原始分 | 惩罚后 | 变化 |
|--------|--------|------|
| 95 | 100 | +5 |
| 90 | 100 | +10 |
| 85 | 93 | +8 |
| 80 | 85 | +5 |
| 75 | 78 | +3 |
| 70 | 70 | 0 |
| 65 | 63 | -2 |
| 60 | 55 | -5 |
| 55 | 48 | -7 |
| 50 | 40 | -10 |

---

## 综合汇总逻辑

```typescript
function computeFinalScore(scores: {
  business: number;
  prompt: number;
  robustness: number;
  safety: number;
  composability: number;
  cost: number;
}, runtime?: number | null): number {
  // 默认路径：D7 未评测（runtimeTrace 缺失），使用 v2.1 权重
  if (runtime === null || runtime === undefined) {
    const rawOverall =
      scores.business * 0.28 +
      scores.prompt * 0.22 +
      scores.robustness * 0.18 +
      scores.safety * 0.14 +
      scores.composability * 0.10 +
      scores.cost * 0.08;

    let finalOverall = applyPenalty(rawOverall);
    if (scores.safety < 60) {
      finalOverall = Math.min(finalOverall, 60);
    }
    return finalOverall;
  }

  // 增强路径：D7 已评测（runtimeTrace 提供），权重重分配
  // 详见 scoring-formulas.md "D7 可选权重分支"
  const rawOverall =
    scores.business * 0.26 +
    scores.prompt * 0.20 +
    scores.robustness * 0.16 +
    scores.safety * 0.12 +
    scores.composability * 0.09 +
    scores.cost * 0.07 +
    runtime * 0.10;

  let finalOverall = applyPenalty(rawOverall);
  if (scores.safety < 60) {
    finalOverall = Math.min(finalOverall, 60);
  }
  return finalOverall;
}
```
