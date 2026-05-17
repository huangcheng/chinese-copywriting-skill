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
- **中文句子内夹英文书籍名/报刊名 → 英文斜体**,不使用中文书名号《》(影响 `WIDTH-CN-PUNCT`)
- **完整的英文整句、英文专有名词内部使用半角标点**(`Stay hungry, stay foolish.`)(影响 `WIDTH-EN-CONTENT`)

若用户在调用时明确要求忽略某条规则,用户指令同样具有优先级 — 在该次会话内生效。

## 不可触及区域

技能**绝不**修改以下内容:

- 围栏代码块 ` ``` ... ``` `
- 行内代码 `` ` ``
- `[文本](URL)` 中的 URL 部分 — 仅可见的「文本」受规则约束
- HTML 标签与属性
- YAML / TOML frontmatter 块

在规则检查之前,把这些区域抽出为占位符;输出修正稿时再原样拼回。

## 三步交互工作流

收到中文文本后,按以下流程执行。每一步都等待用户确认后再前进。

### 第 1 步 — 问题列表

1. 抽出不可触及区域(代码、链接 URL、HTML 标签、frontmatter)为占位符
2. 遍历 `rules.md` 中的每一条规则,扫描剩余文本
3. **在标记之前**应用例外(见上文)
4. 输出违规表:

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

若无违规:输出「未发现问题」,终止流程。

用户响应:
- **y** → 进入第 2 步,带上全部标记项
- **n** → 终止,不做任何修改
- **仅看错误** → 进入第 2 步,过滤掉所有「建议」行

### 第 2 步 — 修改对照(diff)

生成 diff,展示具体改动。Diff 策略(混合):

- **短文本**(原文 ≤ 3 行 **且** 总字符数 ≤ 200):直接在回答中用 ```diff 围栏块手写 `+` / `-` 行
- **长文本**(超过上述任一阈值):写入 `/tmp/cnlint-before.txt` 与 `/tmp/cnlint-after.txt`,执行 `diff -u`,把输出放进围栏块。Bash 路径确保对齐准确;手写 diff 仅用于短到一眼可校对的片段。

短文本示例:

````
```diff
- 在LeanCloud上,数据存储是围绕`AVObject`进行的。
+ 在 LeanCloud 上,数据存储是围绕 `AVObject` 进行的。
```
````

长文本示例命令:

```bash
cat > /tmp/cnlint-before.txt <<'EOF'
<原文>
EOF
cat > /tmp/cnlint-after.txt <<'EOF'
<修正后>
EOF
diff -u /tmp/cnlint-before.txt /tmp/cnlint-after.txt
```

随后提示:

```
是否应用这些修改?
- y          应用全部
- n          取消,不做任何修改
- 1,3,5      仅应用编号 1、3、5
- !4         应用全部,但跳过编号 4
```

用户响应:
- **y** → 应用全部,进入第 3 步
- **n** → 终止,不做任何修改
- **1,3,5** → 仅应用所列编号
- **!4** → 应用全部已标记违规,但跳过编号 4

### 第 3 步 — 输出修正稿

输出最终修正后的文本(围栏块),并附一行总结说明已应用的类别:

```
## 修正稿

\`\`\`
在 LeanCloud 上,数据存储是围绕 `AVObject` 进行的。SSD 一共有 20 TB,角度为 90° 的角是直角。
\`\`\`

已应用:空格(2)、标点(1)、争议(1 / 略过 SUG-LINK-SPACE)
```

## 不在范围内

- 未经用户确认就自动修复
- 中文语法 / 拼写 / 用词检查
- zh-Hans / zh-Hant / English 翻译
- 调用外部工具(pangu.js、autocorrect)— 本技能纯 prompt 逻辑
- 全盘文件扫描 — 仅处理用户在对话中提交的文本
