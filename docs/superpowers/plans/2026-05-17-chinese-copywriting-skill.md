# Chinese Copywriting Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code skill that lints Chinese copy against the 中文文案排版指北 (sparanoid/chinese-copywriting-guidelines), implementing the three-step interactive review workflow defined in `docs/superpowers/specs/2026-05-17-chinese-copywriting-skill-design.md`.

**Architecture:** Two-file skill (SKILL.md + rules.md) stored in this repo under `skill/`, installed at `~/.claude/skills/chinese-copywriting/` via a symlink so updates flow through git. Pure prompt content — no executable code beyond the optional `diff -u` invocation inside the workflow.

**Tech Stack:** Markdown, YAML frontmatter, Bash (`diff -u`), git, gh CLI.

---

## File Structure

Repo root: `/Users/huangcheng/Projects/typology` (= GitHub repo `huangcheng/chinese-copywriting-skill`)

```
typology/
├── docs/superpowers/
│   ├── specs/2026-05-17-chinese-copywriting-skill-design.md  (existing)
│   └── plans/2026-05-17-chinese-copywriting-skill.md         (this file)
└── skill/                                                     (new — symlink target)
    ├── SKILL.md                                              (frontmatter + workflow)
    └── rules.md                                              (12-rule catalog)
```

Install path: `~/.claude/skills/chinese-copywriting → /Users/huangcheng/Projects/typology/skill` (symlink).

**Why `skill/` subdirectory:** Claude Code expects the skill directory to contain `SKILL.md` at its root. Putting the skill in its own subdirectory keeps the repo root from being polluted with skill assets and lets the spec/plan live cleanly under `docs/`.

---

## Task 1: Create skill directory and SKILL.md frontmatter

**Files:**
- Create: `skill/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p /Users/huangcheng/Projects/typology/skill
```

- [ ] **Step 2: Write SKILL.md with frontmatter only**

Write to `/Users/huangcheng/Projects/typology/skill/SKILL.md`:

````markdown
---
name: chinese-copywriting
description: Use when the user wants to review, lint, or proofread Chinese copy for typography and 排版 conformance — checks spacing between 中/英/数字/单位, full-width vs half-width punctuation, proper-noun casing, and informal abbreviations. Triggers on requests like "检查中文排版", "校对这段中文", "看看这段文案有没有问题", or when the user pastes Chinese text and asks for proofreading/lint. Do NOT trigger for general Chinese writing, translation, or grammar tasks.
---

# Chinese Copywriting Linter

A reviewer/linter for Chinese copy. Audits text against the 中文文案排版指北 and walks the user through a three-step interactive correction flow. All user-facing output is in Chinese.
````

- [ ] **Step 3: Verify the frontmatter parses**

Run: `python3 -c "import yaml; f=open('/Users/huangcheng/Projects/typology/skill/SKILL.md').read(); parts=f.split('---'); meta=yaml.safe_load(parts[1]); print('name:', meta['name']); print('desc length:', len(meta['description']))"`

Expected: `name: chinese-copywriting` and a `desc length:` ≥ 200 (description is long).

- [ ] **Step 4: Commit**

```bash
cd /Users/huangcheng/Projects/typology
git add skill/SKILL.md
git commit -m "Add chinese-copywriting skill scaffold and frontmatter"
```

---

## Task 2: Add severity, exceptions, and non-touched-zones sections to SKILL.md

**Files:**
- Modify: `skill/SKILL.md` (append sections)

- [ ] **Step 1: Append severity / exceptions / non-touched-zones sections**

Append to `/Users/huangcheng/Projects/typology/skill/SKILL.md`:

````markdown

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
````

- [ ] **Step 2: Verify the sections were added**

Run: `grep -c "^## " /Users/huangcheng/Projects/typology/skill/SKILL.md`

Expected: `3` (severity, exceptions, non-touched zones)

- [ ] **Step 3: Commit**

```bash
cd /Users/huangcheng/Projects/typology
git add skill/SKILL.md
git commit -m "Add severity model, exception precedence, and protected zones"
```

---

## Task 3: Add the three-step workflow to SKILL.md

**Files:**
- Modify: `skill/SKILL.md` (append workflow)

- [ ] **Step 1: Append the workflow sections**

Append to `/Users/huangcheng/Projects/typology/skill/SKILL.md`:

````markdown

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
````

- [ ] **Step 2: Verify the workflow sections were added**

Run: `grep -c "^### 第" /Users/huangcheng/Projects/typology/skill/SKILL.md`

Expected: `3` (第 1 步, 第 2 步, 第 3 步)

- [ ] **Step 3: Commit**

```bash
cd /Users/huangcheng/Projects/typology
git add skill/SKILL.md
git commit -m "Add three-step interactive review workflow"
```

---

## Task 4: Create rules.md scaffold (legend + TOC)

**Files:**
- Create: `skill/rules.md`

- [ ] **Step 1: Write the rules.md header, legend, and TOC**

Write to `/Users/huangcheng/Projects/typology/skill/rules.md`:

````markdown
# 中文文案排版规则集

来源:[sparanoid/chinese-copywriting-guidelines](https://github.com/sparanoid/chinese-copywriting-guidelines/blob/master/README.zh-Hans.md)

## 图例

| 严重度 | 含义 |
|--------|------|
| **错误** | 指北中的明确规则。默认全部修复。 |
| **建议** | 指北「争议」一节。语法上两种写法都正确,需用户选择是否应用。 |

**例外优先**:若文本匹配指北中记录的例外(产品名按官方写法、°/% 不加空格、海报全角数字、英文书名斜体、英文整句内半角标点),则不视为违规,不出现在报告中。

## 目录

**空格**
- [SP-CN-EN — 中英文之间需要增加空格](#sp-cn-en--中英文之间需要增加空格)
- [SP-CN-NUM — 中文与数字之间需要增加空格](#sp-cn-num--中文与数字之间需要增加空格)
- [SP-NUM-UNIT — 数字与单位之间需要增加空格](#sp-num-unit--数字与单位之间需要增加空格)
- [SP-FW-NOSPACE — 全角标点与其他字符之间不加空格](#sp-fw-nospace--全角标点与其他字符之间不加空格)

**标点**
- [PUNCT-NO-DUP — 不重复使用标点符号](#punct-no-dup--不重复使用标点符号)

**全角/半角**
- [WIDTH-CN-PUNCT — 中文句子使用全角标点](#width-cn-punct--中文句子使用全角标点)
- [WIDTH-NUM-HALF — 数字使用半角字符](#width-num-half--数字使用半角字符)
- [WIDTH-EN-CONTENT — 完整英文整句、专有名词内使用半角标点](#width-en-content--完整英文整句专有名词内使用半角标点)

**名词**
- [NOUN-CASE — 专有名词使用正确大小写](#noun-case--专有名词使用正确大小写)
- [NOUN-NO-SLANG — 不要使用不地道的缩写](#noun-no-slang--不要使用不地道的缩写)

**争议(建议)**
- [SUG-LINK-SPACE — 链接前后增加空格](#sug-link-space--链接前后增加空格)
- [SUG-CORNER-QUOTE — 简体中文使用直角引号](#sug-corner-quote--简体中文使用直角引号)

## 规则
````

- [ ] **Step 2: Verify the TOC has 12 rule entries**

Run: `grep -c "^- \[.*--" /Users/huangcheng/Projects/typology/skill/rules.md`

Expected: `12`

- [ ] **Step 3: Commit**

```bash
cd /Users/huangcheng/Projects/typology
git add skill/rules.md
git commit -m "Add rules.md scaffold with legend and table of contents"
```

---

## Task 5: Add 空格 rules (4 rules)

**Files:**
- Modify: `skill/rules.md` (append rules section)

- [ ] **Step 1: Append the four spacing rules**

Append to `/Users/huangcheng/Projects/typology/skill/rules.md`:

````markdown

### SP-CN-EN — 中英文之间需要增加空格

- **严重度**:错误
- **类别**:空格
- **说明**:中文字符与拉丁字母、数字、行内代码相邻时,二者之间须插入一个半角空格。
- **正确**:在 LeanCloud 上,数据存储是围绕 `AVObject` 进行的。
- **错误**:
  - 在LeanCloud上,数据存储是围绕`AVObject`进行的。
  - 在 LeanCloud上,数据存储是围绕`AVObject` 进行的。
- **例外**:已注册的产品名(如「豆瓣FM」)按官方写法保留,不强制加空格。

### SP-CN-NUM — 中文与数字之间需要增加空格

- **严重度**:错误
- **类别**:空格
- **说明**:中文字符与阿拉伯数字相邻时,二者之间须插入一个半角空格。
- **正确**:今天出去买菜花了 5000 元。
- **错误**:
  - 今天出去买菜花了 5000元。
  - 今天出去买菜花了5000元。
- **例外**:无。

### SP-NUM-UNIT — 数字与单位之间需要增加空格

- **严重度**:错误
- **类别**:空格
- **说明**:数字与单位(Gbps、TB、kg 等)之间须插入一个半角空格。
- **正确**:我家的光纤入屋宽带有 10 Gbps,SSD 一共有 20 TB。
- **错误**:我家的光纤入屋宽带有 10Gbps,SSD 一共有 20TB。
- **例外**:度数(`°`)、百分比(`%`)与数字之间**不加**空格。
  - 正确:角度为 90° 的角,就是直角。新 MacBook Pro 有 15% 的 CPU 性能提升。
  - 错误:角度为 90 ° 的角。新 MacBook Pro 有 15 % 的 CPU 性能提升。

### SP-FW-NOSPACE — 全角标点与其他字符之间不加空格

- **严重度**:错误
- **类别**:空格
- **说明**:中文全角标点符号(,。!?:;「」"")与前后字符之间不应有空格。
- **正确**:刚刚买了一部 iPhone,好开心!
- **错误**:
  - 刚刚买了一部 iPhone ,好开心!
  - 刚刚买了一部 iPhone, 好开心!
- **例外**:无。
````

- [ ] **Step 2: Verify the four spacing rules were added**

Run: `grep -c "^### SP-" /Users/huangcheng/Projects/typology/skill/rules.md`

Expected: `4`

- [ ] **Step 3: Commit**

```bash
cd /Users/huangcheng/Projects/typology
git add skill/rules.md
git commit -m "Add 空格 rules (SP-CN-EN, SP-CN-NUM, SP-NUM-UNIT, SP-FW-NOSPACE)"
```

---

## Task 6: Add 标点 rule (1 rule)

**Files:**
- Modify: `skill/rules.md` (append rule)

- [ ] **Step 1: Append the punctuation rule**

Append to `/Users/huangcheng/Projects/typology/skill/rules.md`:

````markdown

### PUNCT-NO-DUP — 不重复使用标点符号

- **严重度**:错误
- **类别**:标点
- **说明**:同一标点不应连续重复出现。组合标点(`?!`)允许,但同一符号不应叠加。
- **正确**:
  - 德国队竟然战胜了巴西队!
  - 她竟然对你说「喵」?!
- **错误**:
  - 德国队竟然战胜了巴西队!!
  - 德国队竟然战胜了巴西队!!!!!!!!
  - 她竟然对你说「喵」??!!
  - 她竟然对你说「喵」?!?!??!!
- **例外**:无。
````

- [ ] **Step 2: Verify the punctuation rule was added**

Run: `grep -c "^### PUNCT-" /Users/huangcheng/Projects/typology/skill/rules.md`

Expected: `1`

- [ ] **Step 3: Commit**

```bash
cd /Users/huangcheng/Projects/typology
git add skill/rules.md
git commit -m "Add 标点 rule (PUNCT-NO-DUP)"
```

---

## Task 7: Add 全角/半角 rules (3 rules)

**Files:**
- Modify: `skill/rules.md` (append rules)

- [ ] **Step 1: Append the three width rules**

Append to `/Users/huangcheng/Projects/typology/skill/rules.md`:

````markdown

### WIDTH-CN-PUNCT — 中文句子使用全角标点

- **严重度**:错误
- **类别**:全角/半角
- **说明**:中文句子内的标点符号应使用全角(,。!?:;「」())。
- **正确**:
  - 嗨!你知道嘛?今天前台的小妹跟我说「喵」了哎!
  - 核磁共振成像(NMRI)是什么原理都不知道?JFGI!
- **错误**:
  - 嗨! 你知道嘛? 今天前台的小妹跟我说 "喵" 了哎!
  - 嗨!你知道嘛?今天前台的小妹跟我说"喵"了哎!
  - 核磁共振成像 (NMRI) 是什么原理都不知道? JFGI!
  - 核磁共振成像(NMRI)是什么原理都不知道?JFGI!
- **例外**:中文句子内夹有英文书籍名/报刊名时,**不使用**中文书名号《》,应以英文斜体表示(如 *Hackers & Painters*)。

### WIDTH-NUM-HALF — 数字使用半角字符

- **严重度**:错误
- **类别**:全角/半角
- **说明**:数字应使用半角(0-9),不使用全角(０-９)。
- **正确**:这个蛋糕只卖 1000 元。
- **错误**:这个蛋糕只卖 １０００ 元。
- **例外**:在设计稿、宣传海报中如出现极少量数字且需要对齐时,可使用全角数字。

### WIDTH-EN-CONTENT — 完整英文整句、专有名词内使用半角标点

- **严重度**:错误
- **类别**:全角/半角
- **说明**:中文中嵌入的完整英文句子、英文书名、英文专有名词内部,应使用半角标点。
- **正确**:
  - 乔布斯那句话是怎么说的?「Stay hungry, stay foolish.」
  - 推荐你阅读 *Hackers & Painters: Big Ideas from the Computer Age*,非常地有趣。
- **错误**:
  - 乔布斯那句话是怎么说的?「Stay hungry,stay foolish。」
  - 推荐你阅读《Hackers&Painters:Big Ideas from the Computer Age》,非常的有趣。
- **例外**:无。
````

- [ ] **Step 2: Verify the three width rules were added**

Run: `grep -c "^### WIDTH-" /Users/huangcheng/Projects/typology/skill/rules.md`

Expected: `3`

- [ ] **Step 3: Commit**

```bash
cd /Users/huangcheng/Projects/typology
git add skill/rules.md
git commit -m "Add 全角/半角 rules (WIDTH-CN-PUNCT, WIDTH-NUM-HALF, WIDTH-EN-CONTENT)"
```

---

## Task 8: Add 名词 rules (2 rules)

**Files:**
- Modify: `skill/rules.md` (append rules)

- [ ] **Step 1: Append the two noun rules**

Append to `/Users/huangcheng/Projects/typology/skill/rules.md`:

````markdown

### NOUN-CASE — 专有名词使用正确大小写

- **严重度**:错误
- **类别**:名词
- **说明**:品牌名、产品名、公司名按官方写法的大小写格式书写。
- **正确**:
  - 使用 GitHub 登录。
  - 我们的客户有 GitHub、Foursquare、Microsoft Corporation、Google、Facebook, Inc.。
- **错误**:
  - 使用 github / GITHUB / Github / gitHub 登录。
  - 我们的客户有 github、foursquare、microsoft corporation、google、facebook, inc.。
  - 我们的客户有 GITHUB、FOURSQUARE、MICROSOFT CORPORATION、GOOGLE、FACEBOOK, INC.。
  - 我们的客户有 Github、FourSquare、MicroSoft Corporation、Google、FaceBook, Inc.。
- **例外**:网页中为整体视觉风格需要全大写/小写显示时,HTML 内容仍按官方大小写书写,通过 CSS `text-transform: uppercase;` / `text-transform: lowercase;` 控制显示。

### NOUN-NO-SLANG — 不要使用不地道的缩写

- **严重度**:错误
- **类别**:名词
- **说明**:避免使用国内特有、非业界通用的技术名词缩写(Ts、h5、RJS、FED 等)。
- **正确**:我们需要一位熟悉 TypeScript、HTML5,至少理解一种框架(如 React、Next.js)的前端开发者。
- **错误**:我们需要一位熟悉 Ts、h5,至少理解一种框架(如 RJS、nextjs)的 FED。
- **例外**:无。
````

- [ ] **Step 2: Verify the two noun rules were added**

Run: `grep -c "^### NOUN-" /Users/huangcheng/Projects/typology/skill/rules.md`

Expected: `2`

- [ ] **Step 3: Commit**

```bash
cd /Users/huangcheng/Projects/typology
git add skill/rules.md
git commit -m "Add 名词 rules (NOUN-CASE, NOUN-NO-SLANG)"
```

---

## Task 9: Add 争议 rules (2 suggestion rules)

**Files:**
- Modify: `skill/rules.md` (append rules)

- [ ] **Step 1: Append the two suggestion rules**

Append to `/Users/huangcheng/Projects/typology/skill/rules.md`:

````markdown

### SUG-LINK-SPACE — 链接前后增加空格

- **严重度**:建议
- **类别**:争议
- **说明**:中文与链接相邻时,在链接前后插入半角空格更便于阅读。指北将此列为有争议的写法,两种形式语法上均正确。
- **用法**:
  - 请 [提交一个 issue](#) 并分配给相关同事。
  - 访问我们网站的最新动态,请 [点击这里](#) 进行订阅!
- **对比用法**:
  - 请[提交一个 issue](#)并分配给相关同事。
  - 访问我们网站的最新动态,请[点击这里](#)进行订阅!
- **例外**:无。

### SUG-CORNER-QUOTE — 简体中文使用直角引号

- **严重度**:建议
- **类别**:争议
- **说明**:简体中文环境中,使用直角引号「」『』替代弯引号""''。指北将此列为有争议的写法,两种形式语法上均正确。
- **用法**:「老师,『有条不紊』的『紊』是什么意思?」
- **对比用法**:"老师,'有条不紊'的'紊'是什么意思?"
- **例外**:无。
````

- [ ] **Step 2: Verify the two suggestion rules and total rule count**

Run: `grep -c "^### SUG-" /Users/huangcheng/Projects/typology/skill/rules.md`

Expected: `2`

Then verify total rule count:

Run: `grep -cE "^### (SP|PUNCT|WIDTH|NOUN|SUG)-" /Users/huangcheng/Projects/typology/skill/rules.md`

Expected: `12`

- [ ] **Step 3: Commit**

```bash
cd /Users/huangcheng/Projects/typology
git add skill/rules.md
git commit -m "Add 争议 rules (SUG-LINK-SPACE, SUG-CORNER-QUOTE)"
```

---

## Task 10: Install the skill via symlink

**Files:**
- Create: `~/.claude/skills/chinese-copywriting` (symlink → `/Users/huangcheng/Projects/typology/skill`)

- [ ] **Step 1: Check that no skill is already installed at that path**

Run: `ls -la ~/.claude/skills/chinese-copywriting 2>&1`

Expected: `No such file or directory` (lsd) or `ls: cannot access ...` (ls). If something already exists, stop and ask the user before overwriting.

- [ ] **Step 2: Create the symlink**

```bash
ln -s /Users/huangcheng/Projects/typology/skill ~/.claude/skills/chinese-copywriting
```

- [ ] **Step 3: Verify the symlink resolves correctly**

Run: `ls -la ~/.claude/skills/chinese-copywriting && readlink ~/.claude/skills/chinese-copywriting && ls ~/.claude/skills/chinese-copywriting/`

Expected output includes:
- Symlink target: `/Users/huangcheng/Projects/typology/skill`
- Directory listing showing `SKILL.md` and `rules.md`

- [ ] **Step 4: Verify SKILL.md is readable through the symlink and frontmatter parses**

Run: `python3 -c "import yaml; f=open('/Users/huangcheng/.claude/skills/chinese-copywriting/SKILL.md').read(); parts=f.split('---'); meta=yaml.safe_load(parts[1]); print('name:', meta['name'])"`

Expected: `name: chinese-copywriting`

(No commit — the symlink lives outside the repo.)

---

## Task 11: Manual smoke test

This step validates the skill end-to-end. Since the skill is prompt content (not code), the test is performed by invoking Claude Code interactively. The plan engineer should report the observed behavior so any gaps can be fixed.

- [ ] **Step 1: Open a fresh Claude Code session**

```bash
cd ~ && claude
```

- [ ] **Step 2: Paste the following sample input verbatim and request a check**

```
帮我检查中文排版:

在LeanCloud上,数据存储是围绕`AVObject`进行的。我家的SSD一共有20TB,角度为 90° 的角是直角。德国队竟然战胜了巴西队!! 她竟然对你说"喵"?! 请[点击这里](#)进行订阅。我们需要一位熟悉Ts、h5的FED。
```

- [ ] **Step 3: Verify the skill auto-triggers and reports correctly**

Expected: Claude invokes the `chinese-copywriting` skill (visible in the session) and outputs a 问题列表 table in the Step-1 format. The table should include:

| # | 严重度 | 规则 | 原文片段 |
|---|--------|----------|----------|
| - | 错误 | SP-CN-EN | 在LeanCloud上 |
| - | 错误 | SP-NUM-UNIT | 20TB |
| - | 错误 | PUNCT-NO-DUP | 巴西队!! |
| - | 错误 | NOUN-NO-SLANG | Ts、h5...FED |
| - | 建议 | SUG-CORNER-QUOTE | "喵" |
| - | 建议 | SUG-LINK-SPACE | 请[点击这里](#)进行 |

The table should **NOT** include `90°` as a violation (covered by the SP-NUM-UNIT exception).

- [ ] **Step 4: Reply `y` and verify Step 2 (diff) is produced**

Expected: Claude generates a `diff` block (inline since the text is short) showing each change, then prompts with the `y / n / 1,3,5 / !4` selection menu.

- [ ] **Step 5: Reply `y` again and verify Step 3 (corrected output) is produced**

Expected: A fenced code block containing the corrected text and a one-line summary listing applied categories.

- [ ] **Step 6: If anything is wrong, capture the issue**

If the skill fails to trigger, mis-categorizes a rule, flags an exception as a violation, or skips the confirmation step, write down what went wrong. Loop back to the relevant task (e.g. tighten the description in Task 1 if trigger fails; clarify the exception text in Task 5/7 if exceptions are missed) and re-test.

(No commit unless changes were made; if changes were made, commit them under the relevant task's message.)

---

## Task 12: Push everything to the remote

**Files:** (none — git operation)

- [ ] **Step 1: Verify everything is committed**

```bash
cd /Users/huangcheng/Projects/typology
git status
```

Expected: `nothing to commit, working tree clean`

- [ ] **Step 2: Verify the log shows the expected sequence of commits**

```bash
git log --oneline
```

Expected (most recent first): the 争议 rules commit, 名词 rules, 全角/半角 rules, 标点 rule, 空格 rules, rules.md scaffold, workflow, severity/exceptions, skill scaffold, then the original spec commit.

- [ ] **Step 3: Push to origin**

```bash
git push origin main
```

Expected: push succeeds, no rejected updates.

- [ ] **Step 4: Verify the remote shows the new files**

```bash
gh repo view huangcheng/chinese-copywriting-skill --web
```

(Opens browser.) Confirm `skill/SKILL.md` and `skill/rules.md` appear in the file tree.

---

## Self-Review

**Spec coverage:**

| Spec section | Covered by |
|--------------|------------|
| 1. Purpose | Task 1 (frontmatter description) |
| 2. Skill metadata | Task 1 (file paths, name); Task 10 (install path) |
| 3. Activation | Task 1 (description triggers) |
| 4. Severity model | Task 2 (severity table) |
| 5. Exception precedence | Task 2 (exception section) + per-rule 例外 fields (Tasks 5–9) |
| 6. Non-touched zones | Task 2 (zones section) |
| 7.1 Step 1 — 问题列表 | Task 3 (workflow section) |
| 7.2 Step 2 — 修改对照 | Task 3 (workflow section with hybrid diff strategy) |
| 7.3 Step 3 — 输出修正稿 | Task 3 (workflow section) |
| 8. rules.md structure | Task 4 (header/legend/TOC); Tasks 5–9 (rule entries) |
| 9. Rule catalog (12 rules) | Tasks 5 (4 空格) + 6 (1 标点) + 7 (3 全角半角) + 8 (2 名词) + 9 (2 争议) = 12 |
| 10. Out of scope | Task 3 (Out-of-scope section in workflow) |
| 11. Success criteria | Task 11 (smoke test exercises all six criteria) |

All spec sections have a corresponding task. No gaps.

**Placeholder scan:** No "TBD", "TODO", or "fill in" markers. Every rule body, every workflow section, every verification command is fully written out.

**Type/name consistency:** Rule IDs (`SP-CN-EN`, `SP-CN-NUM`, etc.) match between the SKILL.md severity section, the rules.md TOC, the rules.md rule headings, and the smoke-test expected output. Skill name `chinese-copywriting` is consistent across SKILL.md frontmatter, install symlink path, and verification commands.

---

**Plan complete and saved to `docs/superpowers/plans/2026-05-17-chinese-copywriting-skill.md`. Two execution options:**

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**
