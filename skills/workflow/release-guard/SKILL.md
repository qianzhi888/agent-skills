---
name: release-guard
description: 发布守卫技能。上线前在生产/发布分支完成：升级版本号 → 打 tag → 构建并归档回滚包，确保每次发布可追溯、可回滚。说"准备上线"、"打tag"、"发布检查"即可触发。
license: MIT
metadata:
  author: qianzhi
  version: "1.0"
---

# 发布守卫（Release Guard）

> 上线三件套：**版本号升级、生产分支打 tag、回滚包归档**。任何一次发布都必须可追溯到 tag、可在 5 分钟内回滚。

## 触发场景

- 用户说"准备上线"、"要发布了"、"打个 tag"、"发布检查"
- 功能验收通过，准备将发布分支上线
- 用户要求预留回滚包

## 输入

| 信息 | 必填 | 说明 |
|------|------|------|
| 生产/发布分支 | 可选 | 默认 main/master/release 分支，不明确时询问用户 |
| 版本号升级幅度 | 可选 | patch/minor/major；未提供则展示当前版本让用户选择 |
| 构建命令 | 可选 | 未提供则从 package.json scripts / Makefile 推断，推断不出则询问 |

---

## 执行流程

### 步骤 1：发布前置检查（不通过则终止）

```bash
git status --short                  # 工作区必须干净
git branch --show-current           # 确认当前分支
git fetch origin && git status -sb  # 必须与远程同步，无 ahead/behind
```

| 检查项 | 不通过处理 |
|--------|-----------|
| 工作区有未提交变更 | 提示先提交或 stash，终止 |
| 不在发布分支上 | 询问用户正确的发布分支 |
| 与远程不同步 | 先 pull/rebase 对齐后再继续 |

### 步骤 2：升级版本号

**自动识别版本文件**（按项目类型）：

| 项目类型 | 版本文件 | 字段 |
|---------|---------|------|
| Node/前端 | `package.json` | `version` |
| Flutter | `pubspec.yaml` | `version: x.y.z+n` |
| uni-app 小程序 | `manifest.json` | `versionName` + `versionCode` |
| Android | `build.gradle` | `versionName` / `versionCode` |
| iOS | `Info.plist` / 工程文件 | `CFBundleShortVersionString` |

```bash
# 读取当前版本
node -p "require('./package.json').version"   # 示例
git tag --sort=-v:refname | head -3           # 最近 3 个 tag，确认版本递增合理性
```

**升级规则**（遵循语义化版本）：
- `patch`：bug 修复（1.2.3 → 1.2.4）
- `minor`：新功能、向下兼容（1.2.3 → 1.3.0）
- `major`：不兼容变更（1.2.3 → 2.0.0）

**向用户确认新版本号**后再写入文件；小程序等含构建号（versionCode/buildNumber）的项目，构建号必须**同步递增**。

**异常处理**：
- ❌ 找不到版本文件 → 询问用户版本管理方式
- ❌ 新版本号 ≤ 最近 tag → 阻止，提示版本号必须递增

### 步骤 3：提交版本变更

```bash
git add -A
git commit -m "chore: release vX.Y.Z"
```

commit message 固定格式 `chore: release vX.Y.Z`，便于后续检索发布点。

### 步骤 4：生产分支打 tag

```bash
# 附注 tag（含发布信息，可追溯）
git tag -a vX.Y.Z -m "Release vX.Y.Z

- 发布分支: {branch}
- 主要内容: {本次发布的需求/修复摘要}
- 回滚方式: 使用归档回滚包 vX.Y.Z，或回滚至 tag {上一个tag}"
```

**tag 命名规范**：与历史 tag 风格保持一致（先 `git tag` 查看存量风格，如 `v1.2.3` / `release-20260817`）。

**异常处理**：
- ❌ tag 已存在 → 绝不允许覆盖，询问用户是版本号错了还是需要新 tag
- ❌ 上一版本 tag 不存在（首次发布） → 回滚方式改为"回滚包归档"

### 步骤 5：构建并归档回滚包（核心）

> 原则：**本次上线包与上一版本回滚包都要在手**。上线的是新包，但回滚靠的是上一版本的可用产物。

1. **执行构建**：

```bash
# 从 package.json scripts / Makefile / 用户指定中确定构建命令
npm run build        # 或 flutter build apk --release / 小程序平台构建
```

2. **归档当前版本产物**（作为未来回滚的依赖 + 本次发布凭证）：

```bash
mkdir -p release-archive/vX.Y.Z
# 将构建产物（dist/、build/app/outputs/ 等）打包归档
zip -r release-archive/vX.Y.Z/artifact_vX.Y.Z_{YYYYMMDD}.zip {构建产物目录}
```

3. **确认上一版本回滚包存在**：

```bash
ls release-archive/   # 检查上一版本目录是否存在且产物完整
```

| 情况 | 处理 |
|------|------|
| ✅ 上一版本回滚包存在 | 记录路径，写入发布报告 |
| ❌ 不存在 | 从上一 tag 重新构建并归档：`git worktree add` 或 `git archive {上一tag}` 检出构建 |

4. **登记发布日志**（追加到 `release-archive/RELEASE_LOG.md`）：

```markdown
## vX.Y.Z（YYYY-MM-DD）
- Tag: vX.Y.Z（commit: {短hash}）
- 发布分支: {branch}
- 主要内容: {摘要}
- 本版本包: release-archive/vX.Y.Z/artifact_vX.Y.Z_{date}.zip
- 回滚包: release-archive/v{上一版本}/artifact_{...}.zip
- 回滚步骤: {部署平台} 重新部署回滚包 / 应用市场回退版本
```

**异常处理**：
- ❌ 构建失败 → 终止发布流程，先修复构建；**禁止**带着失败构建继续打 tag
- ❌ 产物过大无法归档 → 归档产物清单 + 构建命令 + tag，保证可从 tag 重新构建

### 步骤 6：推送分支与 tag

```bash
git push origin {branch}
git push origin vX.Y.Z
```

**必须用户确认后再推送 tag**（tag 推送后即为发布事实）。

**异常处理**：
- ❌ push 被拒（保护分支等） → 展示错误，按团队流程处理（如走 MR）

### 步骤 7：输出发布报告

```markdown
## 🚀 发布准备完成 vX.Y.Z

| 项目 | 结果 |
|------|------|
| 版本号 | {旧} → {新} |
| Tag | vX.Y.Z @ {commit} |
| 发布分支 | {branch}（已推送） |
| 上线包 | release-archive/vX.Y.Z/artifact_xxx.zip |
| 回滚包 | release-archive/v{上一版本}/artifact_xxx.zip ✅ 已确认存在 |

### 回滚预案
1. {部署平台}切换到回滚包 / 重新部署上一版本产物
2. 验证回滚后核心功能
3. 代码侧：`git revert` 或切回上一 tag 拉 hotfix 分支
```

---

## 禁止事项

- ❌ **禁止跳过前置检查**：工作区不干净、分支不同步时绝不继续
- ❌ **禁止覆盖已有 tag**：tag 冲突必须人工介入，不得 `-f` 强制覆盖
- ❌ **禁止无回滚包发布**：上一版本回滚包缺失时必须先补齐再发布
- ❌ **禁止构建失败继续发布**：构建不通过 = 发布流程终止
- ❌ **禁止未经确认推送 tag**：tag 推送即发布事实，必须用户确认
- ❌ **禁止删除历史归档**：release-archive 只增不删，至少保留最近 5 个版本
