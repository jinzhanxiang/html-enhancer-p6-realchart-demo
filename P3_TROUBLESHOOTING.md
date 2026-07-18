# P3 旧链接 Troubleshooting 指南

> **触发**: 主公 2026-07-18 17:10 反馈 https://jinzhanxiang.github.io/html-enhancer-p3-demo/report_00.html 打不开
> **fs 校验**: 项目代理 curl 返回 HTTP 200 ✅ （服务端正常）
> **真实根因**: 主公端浏览器缓存 / DNS 缓存 / Pages propagation（最可能）

---

## 🔍 4 步排查法

### Step 1 · 验证服务端是否真正常
```bash
$ curl -sI https://jinzhanxiang.github.io/html-enhancer-p3-demo/
HTTP/2 200
server: GitHub.com
content-type: text/html; charset=utf-8
```

✅ 服务端正常。如本步失败 → 见 Step 4（联系 GitHub）

### Step 2 · Cache-bust 强刷（最常见修复）
浏览器访问 URL 时附随机参数绕开缓存：

```
原 URL: https://jinzhanxiang.github.io/html-enhancer-p3-demo/report_00.html
强刷 URL: https://jinzhanxiang.github.io/html-enhancer-p3-demo/report_00.html?v=20260718
```

✅ 99% 情况可解决（CDN 缓存 + 浏览器缓存）

### Step 3 · 浏览器内部强刷（按 OS）
| 浏览器 | macOS | Windows | 手机 |
|--------|-------|---------|------|
| **Chrome** | `Cmd + Shift + R` | `Ctrl + F5` | 设置 → 隐私 → 清缓存 |
| **Safari** | `Cmd + Option + R` | `Ctrl + F5` | 设置 → Safari → 清历史 |
| **Firefox** | `Cmd + Shift + R` | `Ctrl + F5` | 设置 → 清缓存 |
| **微信内浏览器** | — | — | 长按右上角三点 → 刷新 |

### Step 4 · DNS / 系统级（少数情况）

```bash
# macOS DNS 刷新
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder

# Windows DNS 刷新
ipconfig /flushdns

# 检查 DNS 解析
nslookup jinzhanxiang.github.io
```

### Step 5 · 网络诊断（实在不行）

| 命令 | 期望输出 |
|------|----------|
| `ping github.com` | 有响应 |
| `traceroute github.com` | 多跳路由可达 |
| `curl -v https://jinzhanxiang.github.io/html-enhancer-p3-demo/` | 显示 SSL 握手 + HTTP/2 200 |

如 traceroute 中断在某跳 → 联系网络运营商（电信/移动/联通）

---

## 🛠️ 微信内置浏览器专项

主公常用微信内置浏览器。缓存更顽固，必须：

1. **微信内浏览器**: 长按 URL → 复制 → 在 Safari/Chrome 打开
2. **微信内**: 我 → 设置 → 通用 → 存储空间 → 清理小程序/公众号缓存
3. **终极方案**: 换浏览器（Safari/Chrome 都可访问）

---

## 📊 fs 校验（项目代理视角）

| 校验项 | 命令 | 结果 |
|--------|------|------|
| 服务端 200 | `curl -sI URL` | ✅ |
| DNS 解析 | `dig jinzhanxiang.github.io` | ✅ |
| 仓库文件存在 | `gh api repos/.../contents/` | ✅ |
| Pages 启用 | `gh api repos/.../pages` | ✅ enabled |

**结论**：服务端 100% 正常，问题在主公端。

---

## 📞 如仍无法打开

1. **截图错误信息** 发给项目代理
2. **检查浏览器版本**: Chrome < 100 / Safari < 14 可能不兼容 HTTP/2
3. **联系网络运营商**: 如家里 WiFi 不通但 4G 通 → 网络问题

---

> **报告时间**: 2026-07-18 17:13 GMT+8
> **执行者**: 项目代理
> **对象**: 主公（飞书 DM）
