---
name: zmn-flutter-session-review
description: 会话结束复盘。分析本次代码变更，提炼最佳实践与踩坑经验，自动更新 .qoder/rules 规则库。说"复盘"、"总结规则"、"review session"或使用`/zmn-flutter-session-review`即可触发，建议智能体开极致使用此skill。
license: MIT
metadata:
  author: zmn-team
  version: "2.0"
---

# 会话结束复盘 & 规则提炼

> 在 Flutter + Android/iOS 原生混编开发会话结束后，基于**全量会话上下文**（需求→代码→用户修正→讨论），提炼核心规则与最佳实践，更新 `.qoder/rules` 规则库。
>
> **双重目标**：① 提升后续代码生成准确性 ② 节省 token 消耗

## 触发场景

- 用户说"复盘"、"总结规则"、"review session"、"提炼经验"
- 一轮功能开发/重构/问题修复完成后
- 用户修正了 AI 生成的代码后

---

## 执行流程

### 步骤 1：收集变更事实（三个来源，优先级递减）

**来源 A（最高优先级）：会话上下文中的用户修正**

回溯完整对话历史，提取以下高价值信号：

| 信号类型 | 识别特征 | 示例 |
|---------|---------|------|
| **用户直接修正代码** | 用户贴出修改后的 diff / 说"我改了" | 标签宽度 56→65、空对象兜底 |
| **用户指出错误** | "这里不对"、"应该用xxx"、"不需要xxx" | HttpManger 已封装 loading |
| **用户补充知识** | "记住xxx"、"还需要用xxx解密" | 地址需用 MobileDecryptor 解密 |
| **用户质疑决策** | "为什么没有第一次就生成正确" | 反映规则缺失或权重不够 |

**来源 B：git 变更**

```bash
# 按优先级依次检查
git diff --cached --stat  # 已暂存
git diff --stat           # 未暂存
git log --oneline -5      # 最近提交
```

**来源 C：变更文件详情**

对变更文件用 `read_file` 读取，重点关注：

| 目录 | 关注点 |
|------|--------|
| `lib/modules/` `lib/modules_b/` | Flutter 业务页面 view.dart / logic.dart |
| `android/` | 原生 Android 代码、build.gradle、Channel |
| `ios/` | 原生 iOS 代码、Podfile、Channel |
| `lib/utils/` | 工具类封装变更 |
| `.qoder/rules/` | 已有规则（避免重复） |

--

### 步骤 2：分析 & 分类

#### 2.1 提炼优先级（信号强度排序）

| 优先级 | 信号 | 说明 |
|--------|------|------|
| **P0 必提** | 用户明确修正了 AI 生成的代码 | 最强信号：说明当前规则缺失或权重不够 |
| **P0 必提** | 用户说"记住xxx"、"以后都要xxx" | 用户显式要求固化的规则 |
| **P1 应提** | 同一模式在会话中出现 ≥2 次 | 重复问题说明不是偶发 |
| **P1 应提** | 涉及项目封装组件/工具的正确用法 | 防止后续误用 |
| **P2 可提** | 新发现的 API/组件/工具用法 | 扩充规则库覆盖面 |
| **P3 跳过** | 纯业务逻辑、一次性代码 | 不具备复用价值 |
| **P3 跳过** | 已有规则完全覆盖且措辞充分 | 无需重复 |

#### 2.2 变更分类 & 目标文件

| 类别 | 判断标准 | 产出目标 |
|------|---------|----------|
| **UI 还原偏差** | Figma 标注 vs 实际代码有差异（间距、宽度、颜色、字体） | `zmn-flutter-figma-ui-best-practices.md` |
| **组件使用错误** | 用了错误的组件或遗漏了项目封装组件 | 对应 widget 规则 或 `zmn-flutter-code-quick.md` |
| **架构/模式问题** | State 管理、生命周期、数据流向不对 | `zmn-flutter-page-generation.md` 或 `zmn-flutter-controller-lifecycle.md` |
| **工具类/网络层误用** | HttpManger、Toast、Repository 等用法不当 | `zmn-flutter-code-quick.md` 关键规则区 |
| **Flutter↔Native 交互** | Channel 定义、原生回调、混编通信模式 | 评估新建或追加到 `zmn-flutter-tools-usage.md` |
| **Android 原生** | Gradle 配置、Activity/Fragment、权限处理 | 评估新建规则文件 |
| **iOS 原生** | Podfile、ViewController、Info.plist 配置 | 评估新建规则文件 |
| **新发现的通用模式** | 可复用代码模式，现有规则未覆盖 | 评估最合适的规则文件 |

---

### 步骤 3：规则写入策略

#### 3.1 写入位置决策树

**原则：优先更新现有规则文件，不轻易新建**

```
这条规则属于哪个领域？
├── UI/Figma 还原 → zmn-flutter-figma-ui-best-practices.md
├── 组件正确用法 → 对应 widget 的 .md 规则文件
├── 通用编码/CRITICAL → zmn-flutter-code-quick.md（关键规则区）
├── 页面模板/架构 → zmn-flutter-page-generation.md
├── 项目工具类 → zmn-flutter-tools-usage.md
├── 公共库工具类 → zmn-flutter-util-*.md
├── Flutter↔Native → zmn-flutter-tools-usage.md 或新建
├── Android 原生 → 评估新建（如 zmn-android-native.md）
├── iOS 原生 → 评估新建（如 zmn-ios-native.md）
└── 多领域交叉 → 拆分到各自文件，速查表统一索引
```

#### 3.2 写入哪个层级？（Token 优化核心）

规则库有两个层级，写入位置直接影响 token 消耗：

| 层级 | 文件 | 加载时机 | Token 成本 | 适合内容 |
|------|------|---------|-----------|----------|
| **always_on** | `zmn-flutter-code-quick.md` 等 | 每次会话都加载 | 高（~8KB） | 高频 CRITICAL 规则、一行速查代码 |
| **model_decision** | 其他 .md 规则文件 | AI 按需加载 | 低（按需） | 详细示例、低频场景、新领域规则 |

**Token 优化决策**：
- 这条规则**每次写代码都会用到**吗？→ 写入 always_on 速查表（一行代码片段）
- 这条规则**只在特定场景用到**？→ 写入 model_decision 规则文件（完整示例）
- 速查表条目**越精简越好**：`| 功能 | 最短可用代码 |`
- 详细规则文件用 ✅/❌ 对比格式，便于快速扫描

#### 3.3 规则条目格式

**详细规则文件**中追加，格式统一：

```markdown
## N. 规则标题（一句话概括）

### ✅ 推荐做法
描述 + 代码示例

### ❌ 避免做法
描述 + 反面代码示例

### 💡 原因
简要说明为什么（1-2 句）
```

**速查表**中追加，格式：`| 名称 | 最精简代码片段 |`

**关键规则区**中追加 CRITICAL 段落，格式：
```markdown
### 🚫 规则名称（CRITICAL）
一句话描述 + ✅ 正确代码 + ❌ 错误代码
```

#### 3.4 同步更新清单

每次写入规则后，检查以下是否需要同步更新：

| 更新项 | 条件 | 操作 |
|--------|------|------|
| `zmn-flutter-code-quick.md` 速查表 | 新规则涉及高频代码片段 | 追加一行到对应表格 |
| `zmn-flutter-code-quick.md` 关键规则区 | 新规则为 CRITICAL 级别 | 追加段落 |
| `config.yaml` description | 修改了 model_decision 规则文件 | 更新关键词 |
| `README.md` 索引 | 新建了规则文件 | 更新目录树和表格 |

---

### 步骤 4：执行变更

**按以下顺序执行**（严格顺序，不可并行）：

1. **`read_file`** → 读取目标规则文件，确认追加位置，检查去重
2. **`search_replace`** → 在详细规则文件中追加新规则条目
3. **`search_replace`** → 更新 `zmn-flutter-code-quick.md` 速查表/关键规则区（如需要）
4. **`search_replace`** → 更新 `config.yaml` description（如需要）
5. **`search_replace`** → 更新 `README.md` 索引（仅新建规则文件时）

**去重检查**（步骤 1 中必须执行）：
- `grep_code` 搜索目标文件中是否已有相同语义的规则
- 如果已有 → 检查是否需要**强化措辞**或**补充示例**
- 如果重复 → 跳过该条

---

### 步骤 5：保存记忆

用 `update_memory` 保存本次提炼结果：

| 参数 | 值 |
|------|----|
| category | `task_summary_experience` |
| title | `{功能名称}开发规则提炼` |
| content | 列出新增/更新的规则 + 原因（不超过 200 字） |
| keywords | 涉及的技术关键词（不超过 5 个） |

---

## 输出格式

执行完毕后，输出以下总结：

```markdown
## 📋 本次复盘结果

### 提炼的规则

| # | 优先级 | 规则 | 写入文件 | 层级 | 类型 |
|---|--------|------|---------|------|------|
| 1 | P0 | 规则描述 | 文件名 | always_on/model_decision | 新增/强化 |
| 2 | ... | ... | ... | ... | ... |

### 变更统计
- 更新规则文件：N 个
- 新增规则条目：N 条
- 更新速查表：N 处
- 保存记忆：N 条
- 预计每次会话节省 token：~N（基于 always_on 新增量估算）

### 💡 关键收获
1. ...
2. ...

### ⏭️ 后续关注
- 需要在后续会话中验证的规则：...
```

---

## 重要约束

1. **只提炼可复用规则**：纯业务逻辑（如某个接口的字段名）不提炼
2. **优先更新现有文件**：不轻易新建规则文件，避免规则碎片化
3. **控制规则粒度**：一条规则 = 一个明确的 do/don't，不要写长篇大论
4. **代码示例必须真实**：从本次实际变更中提取，不要编造
5. **检查规则去重**：写入前先 `grep_code` 搜索目标文件，确认没有重复
6. **Token 意识**：
   - always_on 规则文件（~8KB 总预算），新增条目必须精简
   - 详细说明放 model_decision 文件
   - 速查表一行不超过 80 字符
7. **覆盖混编场景**：不要只关注 Flutter 侧，Android/iOS 原生变更也需要分析
8. **用户修正 > git diff**：用户在会话中明确指出的问题，优先级高于仅从 diff 观察到的模式

## 无需提炼的情况

如果分析后发现本次变更**全部属于以下情况**，直接告知用户"本次无需提炼新规则"并说明原因：

- 纯业务逻辑实现，无通用性
- 所有模式已被现有规则覆盖且措辞充分
- 仅仅是 bug 修复，无规则性问题
- 用户未做任何修正，AI 生成代码一次通过
