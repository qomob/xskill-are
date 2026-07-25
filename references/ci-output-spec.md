# CI 输出格式规范

> 当输入参数 `ciMode: true` 时，xskill-are 额外输出 CI 友好格式，支持嵌入 CI/CD 流水线作为质量门。
>
> **设计原则：** 使用国际开放标准（JUnit XML / exit code），不引入任何第三方专有格式。

---

## 启用条件

| 输入 | 行为 |
|------|------|
| `ciMode: true` | 额外输出 exit code + JUnit XML 文件 |
| `ciMode: false` 或未提供 | 仅输出标准 AIScore + AIReport（默认行为） |

---

## Exit Code 映射

| 条件 | exit code | CI 含义 |
|------|:---------:|---------|
| `overall ≥ 60` 且无 `fatalFlaws` | **0** | 通过（CI 绿） |
| `overall < 60` | **1** | 失败（CI 红） |
| 存在任意 `fatalFlaws`（即使 overall ≥ 60） | **1** | 失败（CI 红，致命缺陷一票否决） |
| 输入不足 / 评测模型不可用 | **2** | 错误（CI 异常，与"失败"区分） |

> **设计：** exit code 2 用于区分"Skill 评测失败"（exit 1）和"评测器自身故障"（exit 2），便于 CI 脚本分流处理。

---

## JUnit XML 输出

### 文件位置

```
{outputDir}/xskill-junit.xml
```

`outputDir` 默认为当前工作目录，可由调用方指定。

### 字段映射

JUnit XML 是国际标准格式（xUnit Test Result Format），由 Jenkins/GitHub Actions/GitLab CI 等普遍支持。xskill-are 将各维度映射为独立 testcase：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites name="xskill-are" tests="{N}" failures="{F}" errors="{E}" time="{T}s">
  <testsuite name="xskill-are.{skillName}" tests="{N}" failures="{F}" errors="{E}">
    <testcase name="D1_business" classname="xskill-are.business" time="{t1}s">
      <!-- D1 通过：无子节点 -->
    </testcase>
    <testcase name="D2_prompt" classname="xskill-are.prompt" time="{t2}s">
    </testcase>
    <testcase name="D3_robustness" classname="xskill-are.robustness" time="{t3}s">
      <failure message="robustness score 45 &lt; 60">红队用例 2/3 失败：{evidence}</failure>
    </testcase>
    <testcase name="D4_safety" classname="xskill-are.safety" time="{t4}s">
    </testcase>
    <testcase name="D5_composability" classname="xskill-are.composability" time="{t5}s">
    </testcase>
    <testcase name="D6_cost" classname="xskill-are.cost" time="{t6}s">
    </testcase>
    <!-- D7 可选 -->
    <testcase name="D7_runtime" classname="xskill-are.runtime" time="{t7}s">
      <!-- D7 NOT_ASSESSED 时 -->
      <skipped message="runtimeTrace not provided" />
    </testcase>
  </testsuite>
</testsuites>
```

### 判定规则

| 维度状态 | JUnit 节点 |
|---------|-----------|
| 分数 ≥ 60 且无致命缺陷 | `<testcase>` 无子节点（通过） |
| 分数 < 60 | `<failure message="...">` 含证据片段 |
| 维度产生 fatalFlaws（D3/D4/D7-fatal） | `<failure>` 且 message 标注 "FATAL: ..." |
| D7 `NOT_ASSESSED` | `<skipped message="...">` |
| 评测器自身错误（如模型不可用） | `<error message="...">` |

### 顶层属性

| 属性 | 取值 |
|------|------|
| `tests` | 启用的维度数（D7 NOT_ASSESSED 仍计入 tests，但用 skipped 标记） |
| `failures` | 分数 < 60 或 fatalFlaws 的维度数 |
| `errors` | 评测器自身错误的维度数（正常应为 0） |
| `time` | 评测总耗时（秒） |

---

## 标准 JSON 输出（不变）

`ciMode` 启用时，标准 `xskill-report.json`（AIScore + AIReport）**仍然输出**，与 JUnit XML 并存。CI 脚本可任选消费方式：

- **快速质量门**：读 exit code
- **结构化分析**：读 `xskill-report.json`
- **CI 仪表板**：读 `xskill-junit.xml`（被 Jenkins/GitHub Actions UI 原生渲染）

---

## 调用示例

```
输入：
{
  "SKILL.md": "...",
  "capabilities": ["code-review"],
  "ciMode": true,
  "outputDir": "./reports"
}

产物：
./reports/xskill-report.json    （AIScore + AIReport）
./reports/xskill-junit.xml      （CI 友好格式）

stdout：
xskill-are 评测完成
  overall: 78 (良好)
  HRR: A
  fatalFlaws: 0
  D7: NOT_ASSESSED (runtimeTrace 未提供)

exit code: 0
```

---

## 边界声明

1. **JUnit XML 仅是格式，不是测试框架** — xskill-are 不依赖 JUnit / pytest / jest 等任何测试框架
2. **不输出 HTML 报告** — HTML 渲染由前端（消费 AIScore + AIReport 的应用）负责，xskill-are 不做 UI
3. **不支持并行评测协调** — 多 Skill 并行评测由 CI 编排器（如 GitHub Actions matrix）负责，xskill-are 单次只评测 1 个 Skill
