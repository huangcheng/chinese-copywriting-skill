# Chinese Copywriting Skill — Design

**Date:** 2026-05-17
**Status:** Draft for review
**Source:** [sparanoid/chinese-copywriting-guidelines (zh-Hans)](https://github.com/sparanoid/chinese-copywriting-guidelines/blob/master/README.zh-Hans.md)

## 1. Purpose

A reviewer/linter skill that checks user-supplied Chinese text against the published 中文文案排版指北, reports violations in Chinese, and walks the user through a confirm-by-confirm correction flow. The skill does not transform text silently and does not act as a general writing assistant — it is a typography-and-排版 audit tool.

## 2. Skill metadata

- **Name:** `chinese-copywriting`
- **Location:** `~/.claude/skills/chinese-copywriting/`
- **Files:**
  - `SKILL.md` — frontmatter, trigger description, severity model, three-step workflow, diff strategy, exception handling
  - `rules.md` — structured ruleset, twelve rules, with ID, severity, description, 正确/错误 examples, 例外
- **Description (drives auto-trigger):**
  > Use when the user wants to review, lint, or proofread Chinese copy for typography and 排版 conformance — checks spacing between 中/英/数字/单位, full-width vs half-width punctuation, proper-noun casing, and informal abbreviations. Triggers on requests like "检查中文排版", "校对这段中文", "看看这段文案有没有问题", or when the user pastes Chinese text and asks for proofreading/lint.

## 3. Activation

Auto-trigger on intent. Claude invokes the skill when the user:

- explicitly asks for Chinese 校对 / 排版检查 / lint / proofread
- pastes Chinese text and asks whether it has problems
- mentions 中文文案排版 or related terms

Claude does NOT invoke this skill for general Chinese writing or translation tasks.

## 4. Severity model

Two levels, both rendered in Chinese:

| 严重度 | 来源 | 处理 |
|--------|------|------|
| **错误** | 指北中的明确规则(空格、标点、全角半角、名词大小写) | 默认全部修复 |
| **建议** | 指北「争议」一节(链接间空格、直角引号) | 列出但需用户选择 |

Each rule has a stable ID (e.g. `SP-CN-EN`, `WIDTH-CN-PUNCT`) so violation reports cite the rule by ID instead of restating the rule each time. IDs are namespaced by category prefix:

- `SP-` 空格类
- `PUNCT-` 标点重复类
- `WIDTH-` 全角/半角类
- `NOUN-` 名词类
- `SUG-` 争议(建议)类

## 5. Exception precedence (top priority)

**Exceptions defined in the source guide override rules. They are hard overrides, not soft preferences.** Before flagging any violation, the skill must check whether the snippet matches a documented exception — if so, it is **not a violation** and must not appear in the report.

Exceptions in scope:

- **产品名按官方写法**:「豆瓣FM」「淘宝Live」等已注册或品牌官方写法保留(影响 `SP-CN-EN`、`NOUN-CASE`)
- **度数/百分比与数字之间不加空格**:`90°`、`15%`(影响 `SP-NUM-UNIT`)
- **海报/设计稿中的全角数字**:为对齐使用全角数字时不视为违反(影响 `WIDTH-NUM-HALF`)
- **中文句子内夹英文书籍名/报刊名 → 英文斜体**,不应使用中文书名号《》(影响 `WIDTH-EN-CONTENT`)
- **完整的英文整句、英文专有名词内部使用半角标点**(`Stay hungry, stay foolish.`)(影响 `WIDTH-CN-PUNCT`)

If the user explicitly tells the skill to ignore a rule for the current session, that user instruction also takes precedence — the skill respects it for that run only.

## 6. Non-touched zones

The skill must **never** modify content inside:

- fenced code blocks ` ``` ... ``` `
- inline code spans `` ` ``
- URLs / link targets in `[文本](URL)` — only the visible 文本 is subject to rules
- HTML tags and attributes
- YAML / TOML frontmatter blocks

These zones are extracted before rule checking and re-spliced into the corrected output unchanged.

## 7. Three-step interactive workflow

All output Chinese-facing.

### Step 1 — 问题列表

After receiving text, Claude:

1. Strips non-touched zones (Section 6) into placeholders
2. Walks each rule in `rules.md` against the remaining text, applying exceptions (Section 5) **before** flagging
3. Renders a violation table:

```
## 检查结果

发现 5 处问题(3 错误 / 2 建议):

| # | 严重度 | 规则 | 位置 | 原文片段 |
|---|--------|----------|------|----------|
| 1 | 错误 | SP-CN-EN | L2 | 在LeanCloud上 |
| 2 | 错误 | SP-NUM-UNIT | L3 | 20TB |
| 3 | 错误 | PUNCT-NO-DUP | L5 | 太棒了!!! |
| 4 | 建议 | SUG-CORNER-QUOTE | L4 | "喵" |
| 5 | 建议 | SUG-LINK-SPACE | L6 | 请[点击](#)进行 |

是否查看具体修改方案?(y / n / 仅看错误)
```

If no violations: report "未发现问题" and stop.

User responses:
- **y** → continue to Step 2 with all flagged items
- **n** → stop, no changes made
- **仅看错误** → continue to Step 2, filter out 建议 rows

### Step 2 — 修改对照(diff)

Claude generates a diff showing exactly what would change. Diff method (hybrid):

- **短文本**(原文 ≤ 3 行 **且** 总字符数 ≤ 200):直接在回答中使用 ```diff 围栏块手写 `+` / `-` 行
- **长文本**(超过上述任一阈值):写入 `/tmp/cnlint-before.txt` 与 `/tmp/cnlint-after.txt`,执行 `diff -u`,把输出放进围栏块。Bash 路径是确保对齐准确的可靠方式;手写 diff 仅用于短到一眼可校对的片段。

Then prompt:

```
是否应用这些修改?
- y          应用全部
- n          取消,不做任何修改
- 1,3,5      仅应用编号 1、3、5
- !4         应用全部,但跳过编号 4
```

User responses:
- **y** → apply all, continue to Step 3
- **n** → stop, no changes made
- **1,3,5** → apply only the listed violation numbers
- **!4** → apply all flagged violations except number 4

### Step 3 — 输出修正稿

Claude outputs the final corrected text in a fenced code block, followed by a one-line summary of which categories were applied:

```
## 修正稿

\`\`\`
在 LeanCloud 上,数据存储是围绕 `AVObject` 进行的。SSD 一共有 20 TB,角度为 90° 的角是直角。
\`\`\`

已应用:空格(2)、标点(1)、争议(1 / 略过 SUG-LINK-SPACE)
```

## 8. `rules.md` structure

Top of file:

- **Legend**: 严重度 含义(错误 / 建议),何时按 例外 跳过
- **目录**: rule IDs linked to anchors

Each rule entry uses identical fields so Claude scans them uniformly:

```markdown
### SP-CN-EN — 中英文之间需要增加空格
- **严重度**: 错误
- **类别**: 空格
- **说明**: 中文字符与拉丁字母、数字、行内代码相邻时,二者之间须插入一个半角空格。
- **正确**: 在 LeanCloud 上,数据存储是围绕 `AVObject` 进行的。
- **错误**: 在LeanCloud上,数据存储是围绕`AVObject`进行的。
- **例外**: 已注册的产品名(如「豆瓣FM」)按官方写法保留。
```

Required fields per rule: **ID — 标题**, **严重度**, **类别**, **说明**, **正确**, **错误**, **例外**(若无则写「无」).

## 9. Rule catalog (12 rules)

| ID | 类别 | 严重度 | 标题 |
|----|------|------|------|
| `SP-CN-EN` | 空格 | 错误 | 中英文之间需要增加空格 |
| `SP-CN-NUM` | 空格 | 错误 | 中文与数字之间需要增加空格 |
| `SP-NUM-UNIT` | 空格 | 错误 | 数字与单位之间需要增加空格(例外:°、%) |
| `SP-FW-NOSPACE` | 空格 | 错误 | 全角标点与其他字符之间不加空格 |
| `PUNCT-NO-DUP` | 标点 | 错误 | 不重复使用标点符号 |
| `WIDTH-CN-PUNCT` | 全角/半角 | 错误 | 中文句子使用全角标点 |
| `WIDTH-NUM-HALF` | 全角/半角 | 错误 | 数字使用半角字符(海报例外) |
| `WIDTH-EN-CONTENT` | 全角/半角 | 错误 | 完整英文整句、专有名词内使用半角标点 |
| `NOUN-CASE` | 名词 | 错误 | 专有名词使用正确大小写(GitHub、Foursquare 等) |
| `NOUN-NO-SLANG` | 名词 | 错误 | 不要使用不地道的缩写(Ts→TypeScript、h5→HTML5) |
| `SUG-LINK-SPACE` | 争议 | 建议 | 链接前后增加空格 |
| `SUG-CORNER-QUOTE` | 争议 | 建议 | 简体中文使用直角引号「」『』 |

## 10. Out of scope

- Auto-fixing without user confirmation
- General Chinese grammar / spelling / 用词 review
- Translation between zh-Hans / zh-Hant / English
- Pulling in or running external tools (pangu.js, autocorrect) — the skill is pure prompt logic
- File-system-wide scans — the skill operates only on text the user supplies in the conversation

## 11. Success criteria

The skill is working correctly when:

1. Pasting Chinese text with mixed 中英文、数字、标点 produces a correctly-categorized violation table
2. Documented exceptions (产品名、°、%、英文书名斜体) are **never** flagged as violations
3. Code blocks, inline code, URLs, and HTML tags pass through unchanged in the corrected output
4. The user can stop the flow at Step 1 or Step 2 and no edits are made
5. Selective application syntax (`1,3,5`, `!4`, `仅看错误`) works as documented
6. All user-facing prose (rules, severity labels, prompts, summaries) is in Chinese
