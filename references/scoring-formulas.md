# 评分公式与 HRR 分级

> 本文件被 SKILL.md 引用，定义综合评分计算、惩罚函数和等级映射。

---

## 综合评分公式

```
rawOverall = business×0.25 + prompt×0.20 + robustness×0.20
           + safety×0.15 + composability×0.10 + cost×0.10

finalOverall = applyPenalty(rawOverall)
if (safety < 60) finalOverall = min(finalOverall, 60)
```

## 非线性惩罚函数

```
function applyPenalty(rawScore):
    if rawScore >= 95: return rawScore
    return max(0, round(100 - (100 - rawScore)^2 / 12))
```

**效果：** 原始 90→92, 80→67, 70→25, 60→0。加速平庸 Skill 下滑。

---

## 评分等级与 HRR 分级

| 综合分 | 等级 | HRR Tier | 人工修改率 |
|--------|------|----------|-----------|
| ≥90 | 卓越 | — | — |
| ≥85 | — | **S** | <15%（生产级，可直接交付） |
| 80-89 | 优秀 | — | — |
| ≥70 | — | **A** | 15-30%（少量修改后可用） |
| 70-79 | 良好 | — | — |
| ≥50 | — | **B** | 30-50%（需较多修改） |
| 60-69 | 合格 | — | — |
| <60 | 不合格 | — | — |
| <50 | — | **C** | >50%（需大幅重写） |
