# 中文文案排版校对 Skill

一个 Claude Code Skill,基于 [sparanoid/chinese-copywriting-guidelines](https://github.com/sparanoid/chinese-copywriting-guidelines) 对中文文案进行排版校对,通过三步交互流程定位问题、展示差异并输出修正稿。

## 简介

写中文文案时,中英文混排、数字与单位、全角与半角标点的细节常常被忽略。这个 Skill 不做翻译、不做语法纠错、也不会未经确认就替你改稿——它只在你主动发起校对时,把文本按指北的 12 条规则扫一遍,把问题列给你,再让你决定改不改。

适用场景:

- 校对博客、公众号、产品文案
- 检查 README、技术文档中的中英文混排
- 团队稿件统一排版风格,降低沟通成本

## 来源

规则全部来自 sparanoid 维护的 [中文文案排版指北 (简体中文版)](https://github.com/sparanoid/chinese-copywriting-guidelines/blob/master/README.zh-Hans.md),已在 [MIT 许可证](./LICENSE) 下使用。

## 安装

1. Clone 仓库到本地任意位置:

   ```bash
   git clone https://github.com/huangcheng/chinese-copywriting-skill.git
   ```

2. 把 `skill/` 目录链接到 Claude Code 的 skills 目录:

   ```bash
   ln -s "$(pwd)/chinese-copywriting-skill/skill" ~/.claude/skills/chinese-copywriting
   ```

3. 在 Claude Code 中确认 `chinese-copywriting` 已出现在可用 skills 列表中即可。

## 使用

直接把中文文本粘到对话里,加一句校对意图(例如「帮我检查中文排版」「校对一下这段」),Skill 会自动触发。

示例输入:

```
帮我检查中文排版:

在LeanCloud上,数据存储是围绕`AVObject`进行的。我家的 SSD 一共有 20TB,德国队竟然战胜了巴西队!!
```

## 工作流程

三步交互,每一步都等你确认再前进:

1. **问题列表** — 列出所有违规,标出严重度(`错误` / `建议`)、规则 ID、位置、原文片段。也支持「仅看错误」过滤掉建议。
2. **修改对照** — 短文本(≤ 3 行 且 ≤ 200 字符)直接在回答里用 `diff` 围栏块展示;长文本走 `diff -u` 输出。支持选择性应用(`y` / `n` / `1,3,5` / `!4`)。
3. **输出修正稿** — 在围栏块里输出最终文本,并附一行总结说明改了哪些类别。

## 规则集

共 12 条规则,分为 5 类:

| 类别 | 规则数 | 说明 |
|------|------|------|
| 空格 | 4 | 中英文之间、中文与数字之间、数字与单位之间需加空格;全角标点与其他字符不加空格 |
| 标点 | 1 | 不重复使用标点符号(`!!!` `???` 等) |
| 全角/半角 | 3 | 中文句子用全角标点;数字用半角;英文整句内部用半角标点 |
| 名词 | 2 | 专有名词大小写按官方写法;不使用 `Ts`、`h5`、`FED` 等不地道的缩写 |
| 争议 | 2 | 链接前后加空格、简体中文使用直角引号「」 — 这两条原指北标注为风格偏好,Skill 以「建议」严重度提示 |

完整规则定义见 [`skill/rules.md`](./skill/rules.md)。

### 例外优先

指北中记录的例外**覆盖**对应规则,Skill 在标记违规前先检查例外:

- 产品名按官方写法保留(如 `豆瓣FM`、`flickr`、`iPhone`)
- 度数 `°`、百分比 `%` 与数字之间**不加**空格
- 海报/设计稿中为对齐使用全角数字
- 中文句子内夹英文书名/报刊名用斜体 `*Title*`,**不用**中文书名号 `《》`
- 中文句子嵌入完整英文整句时,英文内部使用半角标点(如 `Stay hungry, stay foolish.`)

## 不在范围内

- 自动修复(必须用户逐步确认)
- 中文语法、拼写、用词检查
- 中英、繁简互译
- 调用外部工具(如 [pangu.js](https://github.com/vinta/pangu.js)、[autocorrect](https://github.com/huacnlee/autocorrect)) — 本 Skill 是纯 prompt 内容
- 全盘文件扫描 — 只处理对话中提交的文本

## 文件结构

```
chinese-copywriting-skill/
├── README.md           本文件
├── LICENSE             MIT 许可证
├── skill/              ← 这是要链接到 ~/.claude/skills/chinese-copywriting 的目录
│   ├── SKILL.md        前置触发描述、严重度模型、例外规则、三步工作流
│   └── rules.md        12 条规则的完整定义、图例、目录
└── docs/superpowers/   设计稿与实施计划(开发资料,可忽略)
    ├── specs/
    └── plans/
```

## 致谢

- [sparanoid/chinese-copywriting-guidelines](https://github.com/sparanoid/chinese-copywriting-guidelines) — 规则的完整出处
- [Anthropic Claude Code](https://docs.claude.com/en/docs/claude-code) — 运行环境

## 许可证

[MIT](./LICENSE) © 2026 HUANG Cheng
