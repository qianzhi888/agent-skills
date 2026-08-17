---
name: project-rules-bootstrap
description: 项目冷启动扫描技能。刚下载/克隆的项目，快速扫描代码库生成架构框架文档，并提炼开发规则、公共组件库、工具类等 Rules，为后续 AI 编码建立知识底座。说"扫描项目"、"生成项目文档"、"初始化规则库"即可触发。
license: MIT
metadata:
  author: qianzhi
  version: "1.0"
---

# 项目冷启动扫描 & 规则建库

> 目标：让 AI 在接触新项目的**第一小时内**达到"入职一个月同事"的理解水平——清楚架构分层、编码约定、可复用资产，后续生成代码直接遵循项目既有模式，而不是自由发挥。
>
> 与 `zmn-flutter-session-review` 互补：本技能负责**冷启动建库**，session-review 负责**日常增量维护**。

## 触发场景

- 刚 clone/下载一个新项目，准备开始开发前
- 用户说"扫描项目"、"分析这个项目"、"生成项目文档"、"整理规则"
- AI 生成的代码反复不符合项目约定（说明规则库缺失，应补扫）

## 产出物

| 产出 | 默认路径 | 内容 |
|------|---------|------|
| 架构文档 | `docs/ARCHITECTURE.md` | 技术栈、目录结构、模块职责、构建运行命令 |
| 开发规则 | `.qoder/rules/project-rules.md`（或平台对应位置） | 编码约定、命名、分层、网络/状态/路由规范 |
| 组件速查 | `.qoder/rules/components-quick.md` | 公共组件清单 + 一行用法 |
| 工具速查 | `.qoder/rules/utils-quick.md` | 工具类清单 + 关键签名 |

> 路径按目标平台约定调整（Claude Code → CLAUDE.md，Cursor → .cursorrules 等）。

---

## 执行流程

### 阶段 1：技术栈识别

按特征文件判断项目类型，决定后续扫描重点：

| 特征文件 | 项目类型 | 扫描重点 |
|---------|---------|---------|
| `pubspec.yaml` | Flutter | lib/ 分层、状态管理方案、资源注册方式 |
| `manifest.json` + `pages.json` | uni-app | pages/components、条件编译、多端差异 |
| `package.json` | Node/前端 | 框架（react/vue）、scripts、是否 monorepo |
| `build.gradle` | Android | module 划分、依赖版本管理 |
| `go.mod` / `pom.xml` / `requirements.txt` | 后端 | 目录分层、ORM、中间件 |

```bash
ls -a                                   # 根目录全貌
cat package.json 2>/dev/null | head -40 # 依赖与 scripts
```

### 阶段 2：目录结构与模块划分

1. 生成 2-3 层目录树，**每个顶层目录标注职责**
2. 识别模块边界：`pages/` `modules/` `features/` `services/`
3. 抽样 import 语句，判断模块间依赖方向

```bash
find . -maxdepth 3 -type d \
  -not -path '*/node_modules*' -not -path '*/.git*' -not -path '*/dist*'
```

### 阶段 3：约定与规则提炼（核心）

**每条规则必须有源码依据，是提炼不是发明。**

| 提炼对象 | 扫描来源 | 产出规则示例 |
|---------|---------|------------|
| 编码规范 | `.eslintrc*` / `analysis_options.yaml` / `.editorconfig` / `tsconfig` | "单引号、2 空格、strict 空检查开启" |
| 命名约定 | 抽样 5-10 个业务文件归纳 | "页面文件 kebab-case，组件类 PascalCase" |
| 分层架构 | 目录结构 + import 关系 | "view 只依赖 logic，logic 依赖 api，api 依赖 entity" |
| 网络层约定 | 请求封装工具 | "统一走 HttpManger.request，禁止手动 loading" |
| 状态管理 | 依赖清单 + 使用模式 | "GetX：GetView + GetBuilder，不用 setState" |
| 路由约定 | 路由配置文件 | "pages.json 注册，uni.navigateTo 跳转" |
| 样式约定 | 主题/颜色文件 | "禁止硬编码颜色，统一用 colors.dart 常量" |

**提炼方法：先看配置，再用代码验证**——配置说的和代码做的可能不一致；冲突时以代码多数实践为准，并标 ⚠️ 待确认。

### 阶段 4：公共组件与工具类盘点

```bash
ls components/ widgets/ src/components/ 2>/dev/null
ls utils/ helpers/ common/ src/utils/ lib/utils/ 2>/dev/null
```

对每个组件/工具读取源码提取：
- 名称 + 一句话用途
- 关键 API（props / 方法签名）
- 一行用法示例（**从真实调用处提取，不编造**）

**收录标准**：被 ≥2 处引用的才进速查表（真正公共）；仅单处引用的列入架构文档附录即可。

### 阶段 5：规则分层写入

| 层级 | 文件 | 内容 | Token 预算 |
|------|------|------|-----------|
| always_on | 速查表（components/utils/rules-quick） | 一行条目、高频代码片段 | 总量 ≤8KB |
| model_decision | ARCHITECTURE.md + 详细规则 | 完整示例、模块说明 | 不强制 |

速查表条目格式：`| 名称 | 一句话用途 | 最短可用代码 |`

### 阶段 6：汇总确认与写入

先向用户展示扫描摘要，确认后再写文件：

```
📋 扫描摘要
- 技术栈：Vue3 + uni-app + Vite + pnpm
- 模块：2 个页面模块 / 6 个公共组件 / 3 个工具类
- 提炼规则：12 条（其中 ⚠️ 待确认 2 条）
- 产出文件：docs/ARCHITECTURE.md、.qoder/rules/*.md（3 个）
是否写入？
```

**增量模式**：规则文件已存在时 → 只补充缺失条目，不覆盖；与现有条目冲突的标 ⚠️ 由用户裁决。

---

## 架构文档模板

```markdown
# {项目名} 架构文档

## 1. 概览
一句话定位 + 技术栈表（语言/框架/构建/依赖管理/包管理）

## 2. 目录结构
（带职责注释的 tree）

## 3. 模块职责与关系
| 模块 | 职责 | 依赖 | 被依赖 |

## 4. 构建与运行
| 命令 | 用途 |

## 5. 关键约定速览
（一行规则，详见 rules 文件链接）
```

## 禁止事项

- ❌ **禁止编造规则**：每条规则注明源码出处；没发现就写"未发现"
- ❌ **禁止覆盖已有规则**：增量模式只追加或标冲突
- ❌ **禁止速查表收录非公共代码**：组件/工具须被 ≥2 处引用
- ❌ **禁止未经确认写入**：扫描摘要必须用户确认后才落文件
- ❌ **禁止罗列代码**：架构文档写结构与约定，不粘贴大段实现
