# Figma 切图资源导出 - 团队使用指南

> AI 自动完成：Figma 下载 → 统一转 PNG → @3x 缩放 → 压缩 → 注册 r.dart，你只需要提供 Figma 节点。

## 一、环境准备（首次使用）

```bash
# 安装两个依赖工具（已有则跳过）
brew install librsvg pngquant

# 验证安装
which rsvg-convert && echo "✅ rsvg-convert 已安装" || echo "❌ 未安装"
which pngquant && echo "✅ pngquant 已安装" || echo "❌ 未安装"
```

| 工具 | 用途 | 验证命令 |
|------|------|---------|
| librsvg（rsvg-convert） | SVG → PNG，保留透明背景 | `which rsvg-convert` |
| pngquant | PNG 压缩（>30KB 时启用） | `which pngquant` |

## 二、使用方式

### 方式 1：开发页面时自动触发（推荐）

当你提供 Figma 整页设计稿让 AI 开发页面时，AI 会自动：

1. **识别切图需求** — 分析设计稿中的图标、装饰图、复杂图形
2. **代码中先占位** — 用 `Image.asset(R.icXxx, width: 24.w, height: 24.h)` 占位
3. **输出切图清单** — 页面代码完成后列出所有需要的资源：
   ```
   📋 切图清单（需要你提供 Figma 节点）：
   1. icNewFeatureIcon (24x24) - 新功能图标
   2. icBannerBg (375x200) - 顶部横幅背景
   ```
4. **你提供节点** — 在 Figma 中选中对应图层，复制节点链接给 AI
5. **AI 自动处理** — 下载 → 转 PNG → 缩放 → 压缩 → 注册 r.dart

> 此行为由项目规则 `_code-quick.md` 强制检查保证，AI 每次对话都会加载。

### 方式 2：独立调用 `/zmn-flutter-figma-cutout-png` 技能

不在页面开发上下文中时，直接让 AI 处理切图：

```
/figma-export
处理这个图标：https://www.figma.com/design/xxxxx?node-id=7267-1462
```

或者更简单地说：
```
帮我下载这个图标 @https://www.figma.com/design/xxxxx?node-id=7267-1462
命名为 ic_service_warranty
```

AI 会自动走完整流程，处理完后输出：
```
✅ 切图处理完成
- 资源名称：ic_service_warranty
- 源格式：SVG → PNG（rsvg-convert 转换）
- 设计尺寸：18x18 → @3x: 54x54
- 最终大小：1.6KB（无需压缩）
- 项目路径：assets/images/ic_service_warranty.png
- r.dart 常量：R.icServiceWarranty
- 使用方式：Image.asset(R.icServiceWarranty, width: 18.w, height: 18.h)
```

### 方式 3：批量处理

一次提供多个节点，AI 会逐个处理：

```
帮我处理以下切图：
1. 保修图标 @https://www.figma.com/design/xxx?node-id=111-222
2. 安装图标 @https://www.figma.com/design/xxx?node-id=333-444
3. 换新图标 @https://www.figma.com/design/xxx?node-id=555-666
```

批量完成后会输出汇总：
```
📋 批量处理结果（3/3 成功）
1. ✅ ic_service_warranty — 1.6KB — R.icServiceWarranty
2. ✅ ic_service_install — 2.1KB — R.icServiceInstall
3. ✅ ic_service_exchange — 1.8KB — R.icServiceExchange
```

## 三、如何获取 Figma 节点链接

1. 打开 Figma 设计文件
2. **选中**你需要的图标/图片图层（确保选中的是最终需要的完整元素）
3. 右键 → **Copy link to selection**（或快捷键 `⌘L`）
4. 粘贴给 AI 即可

> **注意**：确保选中的是完整的图标图层，而不是图标内部的某个路径/形状。

## 四、AI 自动处理的完整流程

```
Figma 节点 URL
    ↓
① 依赖检查（rsvg-convert、pngquant）
    ↓
② 提取 nodeId，调用 Figma MCP 获取资源
    ↓
③ curl 下载到 /tmp/
    ↓
④ 格式统一转 PNG
   ├─ SVG → rsvg-convert 输出 @3x PNG（保留透明）
   ├─ JPG → sips 转 PNG
   └─ PNG → 直接使用
    ↓
⑤ @3x 缩放（仅非 SVG 大图，SVG 已在上步完成）
    ↓
⑥ 压缩判断
   ├─ ≤ 30KB → 直接使用
   └─ > 30KB → pngquant 压缩（quality=65-80）
    ↓
⑦ 复制到 assets/images/{命名}.png
    ↓
⑧ 注册到 lib/utils/r.dart
    ↓
⑨ 清理 /tmp/ 临时文件
    ↓
✅ 输出使用方式：Image.asset(R.icXxx, width: xx.w, height: xx.h)
```

## 五、命名规范

| 类型 | 前缀 | 文件名示例 | r.dart 常量 |
|------|------|-----------|------------|
| 图标 | `ic_` | `ic_shop_mall.png` | `R.icShopMall` |
| 背景图 | `bg_` | `bg_banner_top.png` | `R.bgBannerTop` |
| 插图 | `img_` | `img_empty_state.png` | `R.imgEmptyState` |

## 六、关键特性

| 特性 | 说明 |
|------|------|
| 统一 PNG 输出 | 导出的 Figma 资源统一为 PNG 格式（SVG/JPG 自动转换） |
| 透明背景保留 | SVG 使用 rsvg-convert 转换，完美保留透明通道 |
| @3x 自动缩放 | 根据设计尺寸自动计算 @3x 分辨率 |
| 智能压缩 | >30KB 自动用 pngquant 压缩，≤30KB 跳过 |
| 异常检测 | 每步都有失败检测，出错时给出原因和建议 |
| 临时文件清理 | 处理完毕自动清理 /tmp/ 下所有临时文件 |

## 七、常见问题

**Q：AI 没有自动识别切图需求怎么办？**
A：直接说 `/zmn-flutter-figma-cutout-png` 或 "帮我处理这个切图"。

**Q：处理后的图片质量不好怎么办？**
A：AI 压缩后会自动预览验证。如果不满意，让它调整 pngquant quality 参数（提高下限，如 75-90）。

**Q：rsvg-convert 或 pngquant 命令未找到？**
A：执行 `brew install librsvg pngquant` 安装。

**Q：下载到的图片是 JPG 格式怎么办？**
A：AI 会自动将 JPG 转为 PNG。项目统一使用 PNG 格式，无需手动处理。

**Q：我只想用已有的图标，不需要走切图流程？**
A：直接写 `Image.asset(R.icXxx, width: 24.w, height: 24.h)`，前提是 r.dart 中已有对应常量。
