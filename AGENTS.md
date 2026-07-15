# ai-inkwell

一个文档与研究报告工作区，**不是代码项目**。所有产出为 HTML 或 Markdown。

## 目录结构

```
inkwell/
├── .github/workflows/        ← GitHub Actions（双仓同步脚本）
├── .codewhale/skills/        ← CodeWhale Skill 定义
├── .gitignore
├── reports/                  ← 研究报告总目录
│   └── <report-name>/        ← 每个研究报告独立子目录
│       ├── <name>.html或者<name>.md   # 报告正文
│       ├── <name>-sanitized.<ext>     # 脱敏版本
│       └── AGENTS.md                  # 该报告的主题、用户需求、格式要求、约束等
├── sources/                  ← 仅存放通过工具拉取/下载的原始文件
│                               （网页源码、PDF、图片等，不能放 AI 整理后的 md）
├── tmp/                      ← 运行过程产生的所有临时文件
├── AGENTS.md                 ← 本文件
├── README.md
└── LICENSE
```

## 工作流

- 先给大纲或目录结构，经用户确认后再展开细节
- 涉及事实、数据、日期等信息时必须搜索验证并引用来源
- 不确定的结论必须主动标注，不要装成已知事实

## HTML 报告生成标准

- 若无用户指定，默认优雅简洁风格，颜色根据内容主题自适配
- **必须**包含动态侧边目录导航：通过 JS 扫描页面内 `<h2>` / `<h3>` 标签
  自动生成，后续修改正文标题无需手动更新目录

## 禁止行为

- 不执行 `npm install`、`pip install` 等任何包管理/构建命令
- 不在 `sources/` 目录中存放 AI 自己整理、归纳、改写后的 md 文件
- 修改报告后不要覆盖原始版本，先另存为新版本
- 不经 inkwell Skill 确认过脱敏信息就直接执行 `git commit` 和 `git push`

## 双仓架构

- **私有仓：** `Angel-Hair/ai-inkwell`（当前仓库）。两台电脑通过 git 同步全量内容，
  包含原始报告和 `sources/` 中的原始素材。
- **公开镜像：** `Angel-Hair/ai-inkwell-public`。每次 push 到 main 时 GitHub Actions 自动生成，
  CI 脚本位于 `.github/workflows/mirror.yml`。公开仓仅含脱敏版本和框架文件。
