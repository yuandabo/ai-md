# Figma MCP + Claude Code 使用原理与限额总结

> 沉淀日期：2026-10-28
> 来源：`/figma:figma-use` 命令原理讨论

## 1. Skill 的工作原理

`/figma:figma-use` 命令本身**不执行任何操作**，本质是**上下文注入**：

```
/figma:figma-use
  → 读取 SKILL.md 文件内容
  → 作为系统提示注入对话
  → Claude 从此「知道」怎么正确写 Figma Plugin API 脚本
```

Skill 本地路径（Figma 官方插件缓存）：

```
C:\Users\yuanbo\.claude\plugins\cache\claude-plugins-official\figma\<version>\skills\figma-use
```

除 `SKILL.md` 外还有 `references/` 目录（gotchas、组件/变量模式、Plugin API 类型定义 `.d.ts` 等），Claude 按需加载，避免撑爆上下文。

## 2. 执行链路

```
用户指令
   ↓
Claude 调用 use_figma 工具（MCP 协议）
   ↓
JavaScript 代码发送到 Figma 插件沙箱执行（访问完整 Plugin API）
   ↓
return 值 JSON 序列化后回传给 Claude
   ↓
Claude 根据返回的节点 ID / 截图决定下一步
```

三层结构：

| 层 | 角色 |
|---|---|
| Skill（SKILL.md） | 纯知识，教 Claude 怎么写代码 |
| MCP server | 提供 `use_figma`、`get_screenshot` 等工具，负责通信 |
| Figma 插件运行时 | 真正执行 JS、操作画布 |

### 脚本执行的硬性规则（踩坑清单）

- **`return` 是唯一输出通道** —— `console.log` 拿不到，`figma.notify` 未实现
- **每次调用是无状态沙箱** —— `figma.currentPage` 每次重置回第一页，切换页面必须 `await figma.setCurrentPageAsync(page)`
- **字体必须先加载** —— 不 `loadFontAsync` 就改文本会抛错
- **脚本原子性** —— 报错的脚本完全不执行，修复后可安全重试
- **分小步执行（≤10 个逻辑操作/次）** —— 一次写太多，报错整段作废
- **多页面任务**拆成 N 个调用并行发出，而不是一个脚本里循环切页

## 3. 读写双向能力

### 读（Figma → 代码）

| 工具 | 用途 |
|---|---|
| `get_design_context` | 设计稿转代码上下文（设计还原核心） |
| `get_screenshot` | 截图节点，Claude「看」设计 |
| `get_metadata` | 结构信息（层级、命名，无视觉细节） |
| `get_variable_defs` | 读取设计变量/tokens |
| `search_design_system` / `get_libraries` | 搜索已发布设计系统组件 |
| `get_figjam` | 读取 FigJam 白板 |

### 写（代码 → Figma）

| 工具 | 用途 |
|---|---|
| `use_figma` | 万能通道，执行任意 Plugin API JS |
| `generate_figma_design` | 从代码/描述生成完整设计稿 |
| `create_new_file` | 新建 Figma / FigJam / Slides 文件 |
| `upload_assets` / `download_assets` | 上传/下载素材 |

## 4. 两种连接方式与限额差异（重点）

### 本地插件直连（Desktop）

```
Claude Code ──▶ 本地 Figma Desktop ──▶ 插件沙箱执行 JS ──▶ 画布
```

- `use_figma` 在本机桌面端插件沙箱执行，走 Plugin API，与普通插件无异
- **基本不消耗云端 MCP 月度用量额度**
- 前提：Figma Desktop 开着，文件可访问且有编辑权限
- 会话可能超时断开，重连即可

### 云端 MCP（remote，如 mcp.figma.com）

```
Claude Code ──▶ Figma 云端 MCP server ──▶ 云 API ──▶ 文件
```

- `get_design_context`、`search_design_system`、`get_libraries` 等走云端
- **受席位约束**：部分工具需要 Dev seat / Full seat（付费席位）
- **受月度用量限额约束**：按席位+档位分配每月调用额度，超出当月限流
- 通用技术限流：高频请求 429、超大节点响应截断（改选更小子节点可解）

### 对照表

| 操作 | 本地连接 | 云端连接 |
|---|---|---|
| `use_figma` 读写 | ✅ 基本不限 | — |
| `get_design_context` 等读取 | 部分走云端 | 受席位 + 月度额度限制 |

## 5. 实用建议

- 日常读写用**本地 Desktop 连接**，几乎不用操心限额
- 高频大批量读取（如整站设计稿逐节点还原）注意月度额度，必要时问管理员
- 查额度：Figma 网页端 Settings → 用量页面
- 具体额度数字 Figma 调整频繁，以官方文档为准
