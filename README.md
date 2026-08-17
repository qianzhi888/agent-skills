# Agent Skills Portfolio

> 一套面向 AI 编程助手（Coding Agent）的**工程化技能库**：把团队最佳实践、研发工作流沉淀为可复用的 Skill，让 AI 按团队的标准干活。

**共 7 个技能**：4 个来自企业级 Flutter 团队的真实生产实践，3 个通用工程技能。

---

## 什么是 Agent Skill？

Skill 是一份给 AI Agent 的"标准作业程序"（Markdown + YAML Frontmatter）。当用户说出触发词（如"提交代码"、"切图"），Agent 加载对应 Skill，**按预定义的流程、规范和检查清单执行任务**，把个人经验变成团队可复制的能力。

```
用户："帮我提交代码"
   ↓ Agent 匹配 skill description
加载 zmn-commit-push/SKILL.md
   ↓ 按流程执行
提取分支编号 → 分析 diff → 组装 commit message → 用户确认 → push
```

## 技能总览

### 📱 Flutter 工程化（企业生产实践）

| 技能 | 解决的问题 | 亮点 |
|------|-----------|------|
| [zmn-flutter-api-integration](skills/flutter/zmn-flutter-api-integration/SKILL.md) | 接口联调重复劳动：实体类、Api 注册、请求方法、Logic 层手写 | 提供接口信息即自动生成 4 层标准代码，区分有/无返回数据两种模式，附真实案例库 |
| [zmn-flutter-figma-cutout-png](skills/flutter/zmn-flutter-figma-cutout-png/SKILL.md) | 设计稿切图流程繁琐：下载、转换、缩放、压缩、注册 | 9 步全自动流水线（Figma API → PNG → @3x → 压缩 → r.dart 注册），含复合节点渲染方案 |
| [zmn-flutter-session-review](skills/flutter/zmn-flutter-session-review/SKILL.md) | AI 重复犯同样的错，经验无法沉淀 | 会话复盘引擎：从用户修正中提炼规则，分级写入规则库，形成"越用越准"的闭环 |

### 🔧 研发工作流

| 技能 | 解决的问题 | 亮点 |
|------|-----------|------|
| [zmn-commit-push](skills/workflow/zmn-commit-push/SKILL.md) | commit message 格式不统一、手动提交流程繁琐 | 从分支名提取需求编号，读 diff 自动总结改动，确认后一键 push，含冲突处理 |
| [smart-code-review](skills/workflow/smart-code-review/SKILL.md) | 代码审查流于形式、全是风格类噪音 | 四维度加权审查（逻辑/安全/性能/可维护），只输出高信号问题，强制验证后再下结论 |

### 🏗️ 工程方法论

| 技能 | 解决的问题 | 亮点 |
|------|-----------|------|
| [systematic-debugging](skills/engineering/systematic-debugging/SKILL.md) | 调 Bug 靠猜、盲改试错 | 五阶段法（复现→定位→根因→修复→验证），禁止未复现修复、禁止表象修复 |
| [tech-decision-adr](skills/engineering/tech-decision-adr/SKILL.md) | 技术选型拍脑袋、决策不可追溯 | 加权对比矩阵 + 事前验尸（Pre-mortem）+ 标准 ADR 文档产出 |

---

## 设计方法论

这套技能库的编写遵循统一的方法论（详见 [Skill 编写指南](docs/skill-authoring-guide.md)）：

1. **触发词驱动**：`description` 中声明自然语言触发词，降低使用门槛
2. **流程步骤化**：复杂任务拆解为编号步骤，每步给出可执行的命令/动作
3. **异常全覆盖**：每个步骤都定义失败分支与处理策略，Agent 不会中途卡死
4. **强制检查点**：关键操作（提交、覆盖、删除）前必须用户确认
5. **正反面示例**：✅/❌ 对比格式，明确边界，减少 Agent 发挥空间
6. **禁止事项兜底**：每个技能以"禁止事项"收尾，划定安全红线

## 快速开始

将目标技能目录复制到你的 Agent 技能目录即可（以 Qoder 为例）：

```bash
# 复制单个技能
cp -r skills/workflow/smart-code-review ~/.qoder/skills/
# 或项目级
cp -r skills/workflow/smart-code-review /your/project/.qoder/skills/
```

其他平台（Claude Code、Cursor 等）支持 Markdown Skill 格式的均可直接复用，按平台约定调整 Frontmatter 即可。

> 注：`zmn-flutter-*` 系列中的项目专属约定（如 `r.dart`、`HttpManger`）是真实团队规范的体现，迁移到其他项目时替换为对应约定即可。

## 目录结构

```
agent-skills/
├── README.md                     # 本文件
├── showcase.html                 # 技能作品集展示页（浏览器直接打开）
├── docs/
│   └── skill-authoring-guide.md  # Skill 编写方法论
├── templates/
│   └── SKILL.template.md         # 新技能模板
└── skills/
    ├── flutter/                  # Flutter 工程化（3 个）
    ├── workflow/                 # 研发工作流（2 个）
    └── engineering/              # 工程方法论（2 个）
```

## License

[MIT](LICENSE) © qianzhi

## 在线展示

🔗 **GitHub Pages**: https://qianzhi888.github.io/agent-skills/
