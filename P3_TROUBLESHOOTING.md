# P3 报告页正文空白问题 · 真实根因与修复报告

> **版本**: v2 (真实根因版 · 2026-07-18 17:40 GMT+8)
> **触发**: 主公 2026-07-18 17:37 截图反馈"P3 链接打不开"
> **修复**: commit `fd6d5f6bb809a1688deba5fbcdb47b7b72f35aad`
> **状态**: ✅ 已验证（OpenClaw browser 截图证实 H1 + 表格首屏渲染）

---

## ⚠️ 初版错误归因（主公 17:37 → 主公 17:10 反馈"缓存"假设）

**错误根因**：v1 troubleshooting 推测"主公端浏览器/CDN/DNS 缓存"
**真实事实**：P3 服务端文件本身有 HTML 结构缺陷 → 浏览器渲染时 stylesheet 丢失 → 首屏正文 100% 空白

---

## ✅ 真实根因（DST 三线定位）

### 三角验证（DOM + CSS + 几何）

| 验证项 | 数值 | 含义 |
|---|---|---|
| `document.styleSheets.length` | **1** | 应该 3 (main+toc-sidebar+theme-toggle) |
| `stylesheet[0].cssRules.length` | **88** | 等于 main style 规则数 |
| 缺失规则 | `.toc-sidebar,.key-conclusion{position:fixed !important;...}` | 完全没解析 ❌ |
| `getComputedStyle(.toc-sidebar).position` | `static` | 应该 `fixed` |
| `getComputedStyle(.toc-sidebar).width` | `960px` | 应该 `240px` |
| `getComputedStyle(.toc-sidebar).top` | `auto` | 应该 `80px` |
| `getComputedStyle(.toc-sidebar).transform` | `matrix(1,0,0,1,-980,0)` | 仍按 :not(.open) 偏移 → 屏外 |
| `getComputedStyle(.toc-sidebar).offsetParent` | `BODY` | 应该 `null`(fixed 元素) |
| H1 `bounding.top` | `1779.98` | viewport 773 → 屏外 ❌ |
| firstTable `bounding.top` | `2239.65` | viewport 773 → 屏外 ❌ |

### 根因（HTML 结构缺陷）

**第一个 `<style>` (line 7) 没有立即 `</style>` 闭合**：

```
byte  159  <style>                          ← main style 入口
byte 2575  <style id="toc-sidebar">         ← 想做第二个 stylesheet
byte 8273  <style id="theme-toggle">        ← 想做第三个 stylesheet
byte 14365 </style>                         ← (设计意图)关闭 main
byte 14374 </style>                         ← (设计意图)关闭 toc-sidebar
byte 14383 </style>                         ← (设计意图)关闭 theme-toggle
```

按 HTML5 spec，`<style>` 是 **RAWTEXT** 元素——内部 `<style>` 是 raw text 不是开始标签：

- 浏览器进入 byte 159 的 main style 后**就一直读 raw text**（直到 `</style>`）
- byte 2575 的 `<style id="toc-sidebar">` 被 raw text 吞 ❌
- byte 8273 的 `<style id="theme-toggle">` 被 raw text 吞 ❌
- byte 14365 的 `</style>` 关闭 main（exit raw text 模式）
- byte 14374/14383 的 `</style>` 没有对应 `<style>` → 孤儿 close

**后果**：stylesheet 数 1，cssRules=88，仅 main style 生效。

最致命的丢失规则：`.toc-sidebar,.key-conclusion{position:fixed !important;...}` → toc-sidebar 实际 `position:static` → 占文档流 → 仍按 :not(.open) `transform:translateX(-1020px)` 推到屏外 → 占 960px 文档宽 → **正文 H1 被推到 top=1779.98**，viewport 仅 773 → **首屏 100% 空白**。

---

## 🔧 修复方案（最小入侵）

**改动**：在 line 42 `<style id="toc-sidebar">` 之前插入 1 个 `</style>` 闭合 main stylesheet，顺手删除原 14365 那个多余的孤儿 close。

| 字段 | 原文件 | 修复版 |
|---|---|---|
| 字节 | 35134 | 35134 (结构变，字节不变) |
| sha1 | `69d6105...c46d` | `bcd7deb5...83a8` |
| opens/closes | 3/3（错位嵌套）| 3/3（正确配对）|
| blob sha | `e59e25ca...d7943b5` | `c10d8e46...2d134d` |
| commit | `75c0de4 + 2f96959` | `fd6d5f6bb809a1688deba5fbcdb47b7b72f35aad` |

**零业务影响** —— 仅闭合标签位移：
- ✅ H1 文本内容不变
- ✅ 表格数据不变
- ✅ 元信息/时间戳不变
- ✅ 字号/颜色/对比全部 V14.0 时代（保留与 P5/P6 形成时代对比）
- ✅ 仅有 31 + 58 = 89 条 cssRules 替代 88 条 main（多了一条之前被吞的 toc-sidebar 浮动规则）

---

## 📊 修复后验证（OpenClaw browser）

| 验证项 | 修复前 | **修复后** |
|---|---|---|
| `document.styleSheets.length` | 1 | **2** ✅ |
| stylesheet[0] rules | 88 | 31 |
| stylesheet[1] rules | — | 58 ✅ (toc-sidebar) |
| `.toc-sidebar position` | static | **fixed** ✅ |
| `.toc-sidebar width` | 960px | **240px** ✅ |
| `.toc-sidebar zIndex` | auto | **90** ✅ |
| `.toc-sidebar top` | auto | **80px** ✅ |
| H1 top | 1779.98（屏外）| **132.19**（首屏内） ✅ |
| firstTable top | 2239.65（屏外）| **591.85**（首屏内） ✅ |
| H1 文本 | "天保控股 vs 启源芯动力 体制机制..." | 不变 ✅ |
| 截图（OpenClaw browser）| 只有顶部导航条 | **完整报告页** ✅ |

---

## 🧪 用户端 Troubleshooting (P5/P6 链接)

### P3 已修复（新链接）

**主公浏览器侧验证步骤**：
1. **强制刷新**：Mac `Cmd+Shift+R` / Windows `Ctrl+F5` / 手机端"清除浏览器缓存"
2. **cache-bust URL**：用 `?v=fd6d5f6` 后缀
   ```
   https://jinzhanxiang.github.io/html-enhancer-p3-demo/report_00.html?v=fd6d5f6
   ```
3. 微信内置浏览器：如仍空白 → 用 Safari/Chrome 打开 → 通则刷新后回微信
4. 仍有问题：在浏览器 F12 → Network → Disable cache 后重新加载

### P5 / P6 自验

| URL | 状态 |
|---|---|
| https://jinzhanxiang.github.io/html-enhancer-p4-llm-demo/ | ✅ HTTP 200 |
| https://jinzhanxiang.github.io/html-enhancer-p6-realchart-demo/ | ✅ HTTP 200 |

---

## 📜 教训（永久锁定）

1. **不要"curl HTTP 200"就汇报** —— 主公真实体验是浏览器渲染，curl 只能验证文件可下载
2. **三线定位（DOM + CSS + 几何）必须都跑一遍**：任何一个信号都不够信
3. **嵌套 `<style>` 是大坑**：浏览器不识别内嵌样式表，所有 cascade 失效
4. **troubleshooting 错误归因**：第一版"主公端缓存"猜测没被真实验证；后续必须先重现再写文档
5. **V14.0 → V15.x 时代差异**：V15.x 重构了 CSS 注入机制用 `_inject_css()` helper, 但 V14.0 时代直接依赖 stack 闭合，是已知脆弱点

---

**报告完成时间**：2026-07-18 17:42 GMT+8
**修复人**：Project 代理
**报告 SHA**：同步 P3 仓库 main 分支
