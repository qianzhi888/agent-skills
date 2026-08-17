---
name: zmn-flutter-figma-cutout-png
description: Figma 切图导出。提供 Figma 节点链接，自动完成下载 → PNG 转换 → @3x 缩放 → 压缩 → 放入项目 → 注册 r.dart 全流程。说"切图"、"下载图标"、"导出资源"即可触发。
license: MIT
metadata:
  author: zmn-team
  version: "2.1"
---

# Figma 切图资源导出

> 提供 Figma 节点链接，AI 自动完成：下载 → PNG 统一格式 → @3x 缩放 → 压缩（>30KB）→ 注册 r.dart。

## 输入

提供以下任意一种即可开始：

- Figma 节点链接（含 `node-id` 参数）
- 直接说"切图"、"下载图标"、"获取图片资源"、"导出资源"
- 多个节点链接（批量处理）

**如未提供节点链接**，询问用户：
> "请在 Figma 中选中目标图层，右键 → Copy link to selection（或 ⌘L），将链接粘贴给我。"

| 信息 | 必填 | 说明 |
|------|------|------|
| Figma 节点 | ✅ | URL（含 node-id）或直接提供 nodeId |
| 资源名称 | 可选 | 不提供则 AI 根据内容自动命名（snake_case） |
| 设计尺寸 | 可选 | 不提供则从 Figma 设计信息中读取 |

## 前置依赖检查（每次会话首次执行时）

```bash
# 一次性检查所有依赖
echo "--- 依赖检查 ---" && \
which rsvg-convert && echo "✅ rsvg-convert" || echo "❌ rsvg-convert 未安装" && \
which pngquant && echo "✅ pngquant" || echo "❌ pngquant 未安装"
```

| 工具 | 用途 | 安装命令 |
|------|------|---------|
| rsvg-convert | SVG → PNG（保留透明背景） | `brew install librsvg` |
| pngquant | PNG 有损压缩 | `brew install pngquant` |
| curl / sips | 下载 / 图片缩放 | macOS 内置 |
| Figma MCP | 获取设计资源 | Figma Desktop + MCP 插件 |

**缺少任何工具时，必须提示用户安装后再继续，不得跳过。**

快速安装：`brew install librsvg pngquant`

---

## 执行流程

### 步骤 0（首选）：Figma REST API 直接导出

**这是最可靠、质量最高的方式，应始终优先使用。** 通过 Figma REST API 直接导出节点为 @3x PNG，由 Figma 服务端渲染，质量与 Figma 客户端手动导出完全一致。

**前置条件**：需要 Figma Personal Access Token（环境变量 `$FIGMA_TOKEN`，已配置在 `~/.zshrc` 中。若未配置，获取方式：Figma → 头像 → Settings → Personal access tokens → Generate new token）

```bash
# 1. 从 URL 提取 fileKey 和 nodeId
# URL: https://www.figma.com/design/{fileKey}/xxx?node-id={nodeId}
# nodeId 格式转换: URL 中 7267-1462 → API 中 7267:6232

# 2. 调用 Figma Images API 获取导出 URL
curl -s -H "X-Figma-Token: $FIGMA_TOKEN" \
  "https://api.figma.com/v1/images/{fileKey}?ids={nodeId}&scale=3&format=png"
# 返回: {"images": {"nodeId": "https://figma-alpha-api.s3.us-west-2.amazonaws.com/images/xxx"}}

# 3. 下载导出的 PNG
curl -s -f -o /tmp/{资源名}.png "{上一步返回的URL}"

# 4. 验证
file /tmp/{资源名}.png         # 确认为 PNG RGBA
sips -g pixelWidth -g pixelHeight /tmp/{资源名}.png  # 确认为 @3x 尺寸
```

**优势**：
- ✅ Figma 服务端渲染，质量与客户端导出完全一致
- ✅ 自动处理复合节点（多层 SVG 合成），无需手动拼接
- ✅ 保留透明通道（RGBA）
- ✅ 一个 API 调用即可完成，无需 rsvg-convert 等本地工具

**异常处理**：
- ❌ 返回 `{"err":"Invalid token"}` → Token 无效或过期，请用户重新生成
- ❌ 返回 `{"err":"Not found"}` → fileKey 或 nodeId 错误，检查 URL 提取
- ❌ 导出 URL 下载失败 → URL 有时效性，重新调用 API 获取

**如果有 Token，直接执行此步骤，跳过步骤 1-4，直接到步骤 5（压缩判断）。**

---

### 步骤 1（备选）：提取 nodeId 并获取资源信息

> 仅在无法使用 Figma REST API（无 Token）时使用此备选流程。

从 Figma URL 中提取 nodeId（`3341-4305` → `3341:4305`）。

**同时**调用两个 Figma MCP 工具：
- `get_design_context` — 获取资源 URL、设计尺寸、格式信息
- `get_screenshot` — 查看资源视觉效果，确认是目标元素

```
nodeId 提取规则：
URL: https://www.figma.com/design/xxx?node-id=7267-1462
nodeId: 7267:1462  （连字符 → 冒号）
```

**异常处理**：
- ❌ URL 中无 `node-id` 参数 → 提示用户提供正确的 Figma 节点链接
- ❌ MCP 调用失败 → 检查 Figma Desktop 是否打开、MCP 插件是否运行
- ❌ 返回数据中无资源 URL → 节点可能是容器而非图片元素，提示用户选中具体图层

### 步骤 2：下载资源

从 `get_design_context` 返回中提取资源 URL（通常为 `http://localhost:xxxx/assets/...`）。

```bash
curl -s -f -o /tmp/{资源名}.{ext} "{资源URL}"
```

> `-f` 参数：HTTP 错误时返回非零退出码，便于检测下载失败。

下载后**立即验证**：

```bash
# 检查文件是否存在且非空
ls -lh /tmp/{资源名}.{ext}

# 验证文件真实类型（防止扩展名与实际格式不符）
file /tmp/{资源名}.{ext}
```

**异常处理**：
- ❌ curl 返回非零 → 资源 URL 可能已过期，重新调用 `get_design_context` 获取
- ❌ 文件大小为 0 → 下载失败，重试一次
- ❌ `file` 命令显示非图片类型（如 HTML）→ URL 错误，重新提取

### 步骤 3：格式转换 → 统一输出 PNG

**核心规则：通过此技能导出的 Figma 资源，统一输出为 PNG 格式。**

根据下载文件的**实际格式**（以 `file` 命令结果为准，非扩展名），分别处理：

#### 3a. SVG → PNG（rsvg-convert）

```bash
# 从 SVG 的 viewBox 或 width/height 读取设计尺寸，计算 @3x
# 例如 viewBox="0 0 18 18" → @3x: 宽=54 高=54

rsvg-convert -w {3x宽} -h {3x高} /tmp/{资源名}.svg -o /tmp/{资源名}.png
```

**关键说明**：
- `rsvg-convert` 原生支持 PNG Alpha 透明通道，抗锯齿完美保留
- `-w` `-h` 直接输出 @3x 目标尺寸，**无需步骤 4 的 sips 二次缩放**
- ❌ **禁止使用 `qlmanage`**：会丢失透明背景、填充白色底色

**异常处理**：
- ❌ `rsvg-convert` 报错 → 检查 SVG 文件是否完整（`head -5 /tmp/{资源名}.svg` 查看）
- ❌ 输出 PNG 大小为 0 → SVG 可能无有效内容，让用户确认节点选择

#### 3b. JPG/JPEG → PNG（sips 转换）

```bash
sips -s format png /tmp/{资源名}.jpg --out /tmp/{资源名}.png
```

#### 3c. PNG → 无需转换

已是 PNG 格式，跳过此步骤。

**转换后验证**：

```bash
# 确认输出为有效 PNG
file /tmp/{资源名}.png
# 期望输出包含 "PNG image data"

# 查看分辨率
sips -g pixelWidth -g pixelHeight /tmp/{资源名}.png
```

**异常处理**：
- ❌ `file` 不显示 "PNG image data" → 转换失败，检查源文件
- ❌ 分辨率为 0x0 → 文件损坏，重新下载

### 步骤 4：@3x 缩放（仅非 SVG 的大尺寸图片）

> **SVG 已在步骤 3a 直接输出 @3x 尺寸，跳过此步骤。**

对于 PNG/JPG 源图，检查当前分辨率是否大于 @3x（设计尺寸 × 3）：

```bash
# 读取当前分辨率
sips -g pixelWidth -g pixelHeight /tmp/{资源名}.png

# 如果大于 @3x，缩小到目标尺寸
# 设计尺寸 101x219 → @3x = 303x657
sips -z 657 303 /tmp/{资源名}.png --out /tmp/{资源名}_3x.png
```

| 当前分辨率 vs @3x | 操作 |
|-------------------|------|
| 大于 @3x | sips 缩小到 @3x |
| 等于或小于 @3x | 跳过，直接使用 |

**异常处理**：
- ❌ sips 报错 → 文件可能损坏，通过 `read_file` 查看图片确认

### 步骤 5：压缩判断

```bash
# 获取精确文件大小（字节）
stat -f%z /tmp/{最终文件}.png
```

| 大小 | 操作 |
|------|------|
| ≤ 30KB（30720 字节） | 直接使用，跳到步骤 7 |
| > 30KB | 执行步骤 6 压缩 |

### 步骤 6：PNG 压缩（仅 >30KB）

```bash
pngquant --quality=65-80 --speed 1 --force --output /tmp/{资源名}_compressed.png /tmp/{最终文件}.png
```

压缩后**必须验证**：

```bash
# 检查压缩后大小
ls -lh /tmp/{资源名}_compressed.png

# 视觉质量验证（通过 read_file 查看图片）
```

通过 `read_file` 工具查看压缩后的图片，确认视觉质量无明显损失。

**异常处理**：
- ❌ pngquant 返回 99（质量不达标）→ 放宽参数：`--quality=50-90`
- ❌ 压缩后文件反而更大 → 使用压缩前的文件（跳过压缩）
- ❌ 压缩后视觉质量明显下降 → 提高 quality 下限（如 `--quality=75-90`），重新压缩

### 步骤 7：放入项目

```bash
cp /tmp/{最终文件}.png {项目根目录}/assets/images/{命名}.png
```

**命名规范**（全小写 + 下划线）：

| 类型 | 前缀 | 文件名示例 | r.dart 常量 |
|------|------|-----------|------------|
| 图标 | `ic_` | `ic_shop_mall.png` | `R.icShopMall` |
| 背景图 | `bg_` | `bg_banner_top.png` | `R.bgBannerTop` |
| 插图 | `img_` | `img_empty_state.png` | `R.imgEmptyState` |

**放入后验证**：

```bash
# 确认文件已存在于项目目录
ls -lh {项目根目录}/assets/images/{命名}.png
```

**异常处理**：
- ❌ 目标路径已存在同名文件 → 提示用户确认是否覆盖
- ❌ 复制失败 → 检查 assets/images/ 目录是否存在

### 步骤 8：注册 r.dart

在 `lib/utils/r.dart` 文件的类定义末尾（`}` 之前）添加常量：

```dart
/// 资源中文描述
static const String icXxxYyy = 'assets/images/ic_xxx_yyy.png';
```

**注册后验证**：
- 确认常量名使用 lowerCamelCase（如 `icShopMall`）
- 确认路径与实际文件名完全一致
- 确认不与已有常量重名（先搜索 r.dart 中是否已有同名定义）

**异常处理**：
- ❌ r.dart 中已存在同名常量 → 提示用户，确认是更新还是重命名
- ❌ 路径不匹配 → 检查文件名拼写

### 步骤 9：清理临时文件

**必须执行**，删除 /tmp/ 下本次产生的所有临时文件：

```bash
rm -f /tmp/{资源名}.svg /tmp/{资源名}.png /tmp/{资源名}.jpg /tmp/{资源名}_3x.png /tmp/{资源名}_compressed.png
```

> `rm -f`：文件不存在时不报错，确保清理命令始终成功。

清理后确认：

```bash
ls /tmp/{资源名}* 2>/dev/null || echo "✅ 临时文件已清理"
```

---

## 输出格式

每个资源处理完成后，输出以下标准信息：

```
✅ 切图处理完成
- 资源名称：ic_xxx_yyy
- 源格式：SVG → PNG（rsvg-convert 转换）
- 设计尺寸：24x24 → @3x: 72x72
- 原始大小：196KB
- 最终大小：44KB（pngquant 压缩）
- 项目路径：assets/images/ic_xxx_yyy.png
- r.dart 常量：R.icXxxYyy
- 使用方式：Image.asset(R.icXxxYyy, width: 24.w, height: 24.h)
```

处理失败时，输出：

```
❌ 切图处理失败
- 资源名称：ic_xxx_yyy
- 失败步骤：步骤 3 - SVG 转 PNG
- 失败原因：rsvg-convert 报错，SVG 文件内容不完整
- 建议操作：请在 Figma 中重新选中图层，确认选中的是完整图标而非子路径
```

## 批量处理

多个节点按顺序逐个处理，每个走完整流程。单个失败不阻塞后续资源的处理。

批量完成后，输出汇总：

```
📋 批量处理结果（3/3 成功）
1. ✅ ic_service_warranty — 1.6KB — R.icServiceWarranty
2. ✅ ic_service_install — 2.1KB — R.icServiceInstall
3. ✅ ic_service_exchange — 1.8KB — R.icServiceExchange
```

---

## 复合节点处理（CRITICAL）

当 Figma 节点包含多个子 SVG（如"白色圆角背景 + 内部图标"的复合组件），`get_design_context` 会返回多个 SVG URL 和精确的 CSS 定位信息。**此时禁止手动拼接 SVG 路径**，必须按以下流程处理：

### 判断是否为复合节点

- `get_design_context` 返回了 **2 个以上 SVG URL** → 复合节点
- 节点的 CSS 中包含 `position: absolute; inset: ...` 百分比定位 → 复合节点
- 节点是一个容器（背景 + 图标子元素） → 复合节点

### 正确处理流程

1. **下载所有子 SVG**：从 `get_design_context` 返回的 URL 逐个下载到 `/tmp/`
2. **创建 HTML 渲染文件**：将 `get_design_context` 返回的 React/CSS 代码转换为纯 HTML+CSS，将 img src 替换为本地 SVG 文件路径（`file:///tmp/xxx.svg`），所有尺寸乘以 3（@3x 输出）
3. **浏览器截图**：使用 Browser Agent 打开 HTML 文件并截图
4. **裁剪**：用 Python PIL 从截图左上角裁剪 @3x 尺寸区域（DPR=1 时裁剪目标尺寸，DPR=2 时裁剪 2 倍后缩小）
5. **添加圆角透明遮罩**：用 PIL 创建 rounded_rectangle alpha mask 并应用，确保输出为 RGBA PNG
6. **验证**：通过 `read_file` 查看最终 PNG，与 Figma `get_screenshot` 截图对比

### HTML 模板示例

```html
<!DOCTYPE html>
<html>
<head><style>
  * { margin:0; padding:0; box-sizing:border-box; }
  body { background:transparent; width:{3x宽}px; height:{3x高}px; overflow:hidden; }
  /* 直接从 get_design_context 的 CSS 转换，尺寸 ×3，百分比不变 */
</style></head>
<body>
  <!-- 从 get_design_context 的 HTML 转换，img src 改为 file:///tmp/xxx.svg -->
</body>
</html>
```

### Python 裁剪+圆角模板

```python
from PIL import Image, ImageDraw
img = Image.open('/tmp/screenshot.png')
cropped = img.crop((0, 0, target_w, target_h))  # 从左上角裁剪
cropped = cropped.convert('RGBA')
mask = Image.new('L', cropped.size, 0)
draw = ImageDraw.Draw(mask)
draw.rounded_rectangle([(0,0), (cropped.width-1, cropped.height-1)], radius=圆角*3, fill=255)
cropped.putalpha(mask)
cropped.save('/tmp/final.png')
```

### ❌ 错误做法（曾导致图标变形的教训）

- ❌ **手动拼接 SVG 路径**：从多个子 SVG 中提取 `<path>` 然后用 `<g transform>` 手动组合 → 定位不准确，比例失真
- ❌ **猜测子元素位置**：根据 viewBox 手动计算偏移 → 与 Figma CSS 百分比定位不一致
- ❌ **只下载主 SVG**：复合节点有多个子层，只取其中一个无法还原完整图标

### ✅ 正确做法

- ✅ **使用 Figma CSS 定位**：`get_design_context` 返回的 CSS（百分比 inset、flex 布局、rotate 等）是精确的
- ✅ **HTML+浏览器渲染**：让浏览器精确执行 CSS 布局，截图结果与 Figma 设计完全一致
- ✅ **对比验证**：最终 PNG 必须与 `get_screenshot` 截图视觉一致

---

## 禁止事项

- ❌ **禁止使用 `qlmanage` 转换 SVG**：会丢失透明背景，填充白色底色
- ❌ **禁止输出非 PNG 格式**：项目统一使用 PNG，JPG/SVG 必须转换
- ❌ **禁止跳过临时文件清理**：处理完毕后必须清理 /tmp/ 下的临时文件
- ❌ **禁止跳过质量验证**：压缩后必须通过 read_file 查看图片确认质量
- ❌ **禁止硬编码路径**：图片必须注册到 r.dart，代码中使用 `R.icXxx` 引用
- ❌ **禁止跳过依赖检查**：首次执行必须验证 rsvg-convert 和 pngquant 已安装
- ❌ **禁止手动拼接复合节点的 SVG 路径**：复合节点必须通过 HTML+浏览器渲染处理
