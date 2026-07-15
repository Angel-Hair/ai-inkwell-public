# ai-inkwell

这是我用AI辅助编写研究报告的项目级框架。
> *里面包含了一些我公开的报告文档，内容仅供参考*

## 工作环境

- CodeWhale + VSCode

> *虽然不建议，但你确实可以换成其他 AI Agent 工具。不过为了完整发挥本项目功能，使用时需要显式提醒AI理解项目框架或者进行针对性修改（比如项目内 Skills 是适配 CodeWhale 设计的）*

## 你要如何使用

纯手动（太麻烦，不推荐）：

<details>
<summary> clone 后的一些步骤</summary>

1. 删掉 `reports/` 下我的报告
2. 去 GitHub 新建一个私有仓和一个公开空仓，把 clone 下来的目录关联到私有仓
3. 把 `AGENTS.md`、`.codewhale/skills/inkwell/SKILL.md`、`.github/workflows/mirror.yml` 里的仓库路径、用户名改成你的
4. 在私有仓 Settings → Secrets and variables → Actions 添加 `PUBLIC_REPO_TOKEN`——Fine-grained PAT，给公开仓 Contents 读写权限就行
5. push 到 main，看 Actions 是否跑通

</details>
<br>

或者直接让AI帮你解决（**推荐**）：

`clone` 下来，在目录内打开 CodeWhale ，直接给AI说：

```text
把这个项目改成我自己的，你需要先删除原项目作者的报告并且理解整个项目的机制原理，最后需要告诉我后续在 GitHub 上配置的步骤。
```

通常情况下，AI 会帮你批量替换 `AGENTS.md`、`.codewhale/skills/inkwell/SKILL.md`、`.github/workflows/mirror.yml` 里的名称和路径，删掉原报告，然后输出一份 GitHub 配置清单让你照着做。

## 设计哲学

我之前有很多自己的或者公司的研究报告需要AI辅助编写，并且我是一个很偏执的人，网上买个贵点的东西都需要让AI先给我生成一份"论文"，所以这玩意儿对我来说是刚需。

在过去长久的实践中，我受够了上下班需要大量繁杂步骤来同步进度和环境，并且每次新建文档都需要单独再定制化的给AI喂各种规范和约束。所以我在尝试了许多 AI Agent 后，适配 CodeWhale 写了一个代码级的项目框架，并且产生了 `上传到 GitHub 上 + 公私双仓库` 的想法，这样做的话还能把一些我觉得有价值的报告上传到网上供大家参考。

本项目得益于显式的、轻量化的框架文件约束，以及低耦合的自动化双仓设计，即使直接切换到其他 AI Agent ，也至少能保留 80% 以上的功能。如果配合上 CodeWhale 的 `宪法系统` ，能够在保证手术刀级的软约束下，让AI知道如何规范化的帮助我生成研究报告。

## 项目细节

如果有需要，项目结构和工作流等内容都在 [AGENTS.md](./AGENTS.md) 里。
