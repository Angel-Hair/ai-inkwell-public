# 联网搜索工具使用指南（CodeWhale版）

> 最后更新：2026-07-14 适配 CodeWhale v0.8.67。禁用 web_search / web.run，新增 Playwright MCP 浏览器自动化配置。

## 当前环境

| 工具 | 状态 | 后端 |
|---|---|---|
| `mcp_searxng_web_search` | ✅ 推荐 | SearXNG @ 107.182.25.64:8814，通过 MCP 插件 `@kevinwatt/mcp-server-searxng` 接入。聚合 Google+Bing+Wikipedia+DDG |
| `web_search`（内置） | ❌ 已禁用 | 默认 DDG HTML + Bing fallback。DDG 返回 bot challenge（GitHub Issue #964），Bing fallback 返回垃圾结果，不可用 |
| `fetch_url` | ✅ 可用 | HTTP GET 直连 URL，HTML 剥离为纯文本。不执行 JS，只能抓服务端渲染页面。走 Clash 代理 |
| `web.run`（内置） | ❌ 已禁用 | 搜索端同 `web_search`（受 DDG bot challenge 影响），open/click/find 不执行 JS。功能已被 `fetch_url` + Playwright MCP 全覆盖 |
| `mcp_playwright_*` | ✅ 推荐 | Playwright MCP 接入 Firefox。真实浏览器内核，执行 JS，可过 SPA/反爬。详见下文 |
| `agent` 子代理 | ✅ 可用 | `agent` 工具 + `general` 角色做多步调研 |

## `searxng` MCP详细说明

### 工作机制

```
CodeWhale 桌面端
  └─ MCP 子进程 (npx @kevinwatt/mcp-server-searxng)
       └─ HTTP POST /search (JSON API) → SearXNG @ 107.182.25.64
            └─ 并行查询: Google + Bing + Wikipedia + DuckDuckGo + ...
                 → 聚合、去重 → 返回 JSON 结果
```

注意：MCP 插件使用 SearXNG 的 JSON API（`format=json`），需要在 SearXNG settings.yml 的 `search.formats` 中包含 `json`，否则返回 403。

服务器在海外，上游引擎看到的出口 IP 均为海外 IP，不受中国 IP 搜索降级影响。MCP 子进程通过 `HTTP_PROXY` 环境变量走 Clash 代理访问海外服务器。

### 调用参数建议

调用时建议明确传入 `language`，避免跨语言杂讯和低质结果：

```json
{
  "query": "搜索关键词",
  "language": "zh"
}
```

常用可选参数：`categories`（[`general`/`news`/`science`/`images`/`videos` 等]）、`safesearch`（过滤成人和不适宜内容）、`engines`（搜索引擎）、`time_range`（`day`/`week`/`month`/`year`）、`page`（页码）。

中英文均可正常使用，无需特殊处理。之前的各种规避手段（Google News fetch、中文关键词混用等）不再必要。

### 注意事项

**首次调用可能超时：** SearXNG 实例因上游引擎聚合延迟导致间歇性超时，第一次 `mcp_searxng_web_search` 调用可能返回 "All SearXNG instances failed" 错误，绝大多数情况下第二次调用即可正常返回。

**可能遇到 Clash 代理节点不稳定：**——MCP 子进程通过 `HTTP_PROXY` 走 Clash，代理节点不稳定也会导致超时。

## `fetch_url` 详细说明

### 工作机制

CodeWhale 内置的 `fetch_url` 工具直连目标 URL，抓取文本内容（HTML 自动剥离为纯文本）。不走 SearXNG，仍走 Clash 代理。稳定性取决于目标网站。

### 使用场景

当 URL 已经知道时，`fetch_url` 比 `web_search` 更快——不用绕搜索引擎，直接请求目标 URL。用于：
- 抓取已知文档页面的内容
- 读取 GitHub raw 文件
- 获取 API 响应

注意：`fetch_url` 不执行 JavaScript，只能抓服务端渲染页面。SPA、Cloudflare 保护页面无法获取内容。

## `playwright` MCP详细说明

驱动真实 Firefox 浏览器，完整执行 JavaScript、渲染 DOM。通过 **accessibility tree snapshot** 传递页面结构——比截图+视觉模型方案更快、更省 token，且能精确定位到每个元素的 `ref`。

### 工作机制

```
CodeWhale 桌面端
  └─ MCP 子进程 (npx @playwright/mcp@latest)
       └─ Firefox 浏览器（本地 firefox.exe）
            └─ 通过系统代理（Clash）访问外网
                 页面渲染 → accessibility tree snapshot → 返回 ref 树
```

与 SearXNG 不同，Playwright MCP 不需要显式注入 `HTTP_PROXY`——浏览器直接走系统代理设置。实测出口 IP 为海外地址，确认走 Clash 代理。

经过实测，约 30 个工具中大部分功能正常。核心工具是 `browser_snapshot`——通过 accessibility tree 传递页面结构，每个元素带唯一 `ref` 引用，后续 `browser_click`、`browser_type`、`browser_find`、`browser_evaluate` 等交互都依赖 `ref` 定位。`browser_tabs` 支持多标签管理，`browser_take_screenshot` 可截取视口或全页，`browser_console_messages` 和 `browser_network_requests` 用于诊断页面行为。`browser_drag`、`browser_drop`、`browser_file_upload`、`browser_handle_dialog` 等需特定场景，尚未实测。

### 注意事项

1. **`ref` 仅在当前 snapshot 有效。** 页面跳转、后退后旧 `ref` 全部失效，交互前先 `browser_snapshot` 刷新。
2. **反爬验证。** 真实 Firefox 仍可能触发 bot 检测（如 Google reCAPTCHA、DataDome 滑块）。通过 `browser_snapshot` 检测验证类型后，**文字提示用户手动完成**，通过后继续。AI 无法自动过验证码，不要反复重试——会触发更严厉的封禁。
3. **Cookie 弹窗。** 欧洲站点常弹出 cookie 横幅。若只是顶部横幅不遮挡目标元素，可跳过；若为全屏遮罩层拦截了后续点击，先点掉「Reject all」或「Accept」再继续。
4. **Snapshot 体积。** 复杂页面 snapshot 可能很大，优先用 `browser_find` 定位目标区域，减少全页 snapshot 次数。
5. **浏览器上下文存活。** 长时间闲置或首次打开后可能遇到 "Target page has been closed" 错误，重新 `browser_navigate` 即可。

## `agent` 子代理（深度调研 / 并行搜索）

### 何时使用子代理 vs 内联搜索

| 场景 | 用 `mcp_searxng_web_search` 内联 | 用 `agent` 子代理 |
|------|----------------------------------|-------------------|
| 单一事实查询（"XX 的最新版本是多少"） | ✅ 快、便宜 | ❌ 浪费 |
| 需要对比 3-5 个来源的简短摘要 | ✅ 1-5 次搜索 + 自己总结 | ❌ 启动开销大于收益 |
| 需要覆盖多个独立主题方向 | ⚠️ 可以但要串行多轮 | ✅ 并行开 2-5 个 |
| 深度调研（论文级，需要 5+ 来源/章节） | ❌ 主对话会被大量搜索结果淹没 | ✅ 子代理独立执行，只返回摘要 |
| 需要搜索 + 抓取多个 URL 再综合 | ❌ 多轮交互拉长主对话 | ✅ 子代理内部串行完成 |

**决策规则**：如果任务需要 **≥5 次搜索 + 跨来源综合**，开一个 `general` 子代理。如果只需要 1-5 次搜索，直接在主线用 `mcp_searxng_web_search`。

### 工作机制

子代理继承完整工具注册表（`mcp_searxng_web_search`、`fetch_url` 等）。关键约束：
- **无法中途停止**——必须在 prompt 里预设明确停止条件（约束是建议而非命令，子代理可能无视）。
- **角色选择**：调研用 `general`。`explore` 是代码探索，不应用于 web research。
- **`fork_context`**：`true` 继承主对话上下文（省 token）；默认 `false` 用于纯外部调研。

### 分级约束策略

**原则：必须有停止条件。** 来源数、章节列表、主题范围——至少给一个。深度调研原则上不设字数上限。

| 场景 | 停止条件 | 字数 | 角色 | fork_context |
|------|---------|------|------|-------------|
| 快速（≤3 来源） | 3 来源或 3 次无果搜索→接受 not found | ≤200 | `general` | 默认 |
| 深度（论文级） | 每章节 ≥5 来源，或 3 种搜索策略用尽 | 不限 | `general` | `false` |
| 并行（多方向） | 每方向 5 来源或 3 次失败 | 不限 | `general` | 默认 |

并行调研各子代理独立返回后，主对话交叉比对、合并重复、标注矛盾点。

### ❌ 避免

- **不要开无停止条件的子代理**——`"Tell me everything about X"` 无边界，几乎必跑死。
- **不要让子代理修本地 bug**——`"Find the bug in my auth module"` 子代理收到的只是文字描述，它看不到当前代码状态（除非 `fork_context: true` 且主对话里已有相关文件内容）。让它搜索调研，你来做代码判断。

## 环境部署信息

### SearXNG

```
服务器：Debian 13 x86_64 @ 107.182.25.64
端口：8814（非标准高位端口，防批量扫描）
反向代理：nginx → uwsgi → SearXNG
搜索路径：/search（/searxng 路径已从 nginx 移除）
静态文件：nginx 直接 serve /static/
```

优化配置（`/etc/searxng/settings.yml`）：

| 配置 | 值 | 说明 |
|---|---|---|
| `search.formats` | `["html", "json"]` | **必须包含 json**，否则 MCP 插件返回 403 |
| `limiter` | `false` | 自用不公开，无需限流 |
| `valkey.url` | `false` | 限流已关，Valkey 已卸载 |
| `autocomplete` | `duckduckgo` | 保留搜索建议（人类浏览器用） |
| `favicon_resolver` | `duckduckgo` | 保留网站图标 |
| `image_proxy` | `true` | 保留图片隐私代理 |

防火墙（ufw）：

```
默认：deny incoming, allow outgoing
开放端口：26545 (SSH), 8814 (SearXNG)
```

### CodeWhale 配置

#### 禁用了内置 `web_search` 和 `web.run`

在 `C:\Users\<用户名>\.codewhale\config.toml` 中添加：

```toml
[tools.overrides]
"web_search" = { type = "disabled" }
"web.run" = { type = "disabled" }
```

两者搜索端共享同一后端（DDG HTML + Bing fallback），DDG 对所有 CodeWhale 请求返回 bot challenge，不可用。搜索由 `mcp_searxng_web_search` 替代，页面抓取由 `fetch_url` 承担，JS 渲染由 Playwright MCP 承担。

### Clash Party

```
代理：127.0.0.1:7890
配置路径：%APPDATA%\mihomo-party\
```

## AI 决策流程

```
需要搜索?
  └─ mcp_searxng_web_search ✅ 主力搜索，中英文均走 SearXNG，建议传 language

需要深度调研?
  └─ agent 子代理（general 角色），按分级约束执行
     （子代理内调用 mcp_searxng_web_search + fetch_url）

需要抓取已知 URL（静态页面）?
  └─ fetch_url ✅ 不执行 JS，只能抓服务端渲染页面

需要 JS 渲染 / 截图 / 表单交互?
  └─ Playwright MCP ✅ 真实 Firefox，完整 JS 执行

遇到反爬验证（CAPTCHA / 滑块 / DataDome）?
  └─ browser_snapshot 检测验证类型 → 文字提示用户手动完成 → 通过后继续
```
