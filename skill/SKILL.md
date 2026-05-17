---
name: chinese-copywriting
description: Use when the user wants to review, lint, or proofread Chinese copy for typography and 排版 conformance — checks spacing between 中/英/数字/单位, full-width vs half-width punctuation, proper-noun casing, and informal abbreviations. Triggers on requests like "检查中文排版", "校对这段中文", "看看这段文案有没有问题", or when the user pastes Chinese text and asks for proofreading/lint. Do NOT trigger for general Chinese writing, translation, or grammar tasks.
---

# Chinese Copywriting Linter

A reviewer/linter for Chinese copy. Audits text against the 中文文案排版指北 and walks the user through a three-step interactive correction flow. All user-facing output is in Chinese.

## 严重度模型

两个等级,均以中文呈现:

| 严重度 | 来源 | 默认处理 |
|--------|------|----------|
| **错误** | 指北中的明确规则(空格、标点、全角半角、名词大小写) | 默认全部修复 |
| **建议** | 指北「争议」一节(链接前后空格、直角引号) | 列出但需用户选择 |

每条规则有稳定 ID(如 `SP-CN-EN`、`WIDTH-CN-PUNCT`),违规报告以 ID 引用规则,不必重复规则正文。ID 前缀按类别命名:

- `SP-` 空格类
- `PUNCT-` 标点重复类
- `WIDTH-` 全角/半角类
- `NOUN-` 名词类
- `SUG-` 争议(建议)类

完整规则定义见 `rules.md`。

## 例外优先(最高优先级)

**指北中定义的例外覆盖规则。它们是硬性覆盖,不是软性偏好。** 在标记任何违规之前,必须先检查文本片段是否匹配已记录的例外 — 如果匹配,则**不视为违规**,不得出现在报告中。

适用例外:

- **产品名按官方写法**:「豆瓣FM」「淘宝Live」等已注册或品牌官方写法保留(影响 `SP-CN-EN`、`NOUN-CASE`)
- **度数/百分比与数字之间不加空格**:`90°`、`15%`(影响 `SP-NUM-UNIT`)
- **海报/设计稿中的全角数字**:为对齐使用全角数字时不视为违反(影响 `WIDTH-NUM-HALF`)
- **中文句子内夹英文书籍名/报刊名 → 英文斜体**,不使用中文书名号《》(影响 `WIDTH-EN-CONTENT`)
- **完整的英文整句、英文专有名词内部使用半角标点**(`Stay hungry, stay foolish.`)(影响 `WIDTH-CN-PUNCT`)

若用户在调用时明确要求忽略某条规则,用户指令同样具有优先级 — 在该次会话内生效。

## 不可触及区域

技能**绝不**修改以下内容:

- 围栏代码块 ` ``` ... ``` `
- 行内代码 `` ` ``
- `[文本](URL)` 中的 URL 部分 — 仅可见的「文本」受规则约束
- HTML 标签与属性
- YAML / TOML frontmatter 块

在规则检查之前,把这些区域抽出为占位符;输出修正稿时再原样拼回。
