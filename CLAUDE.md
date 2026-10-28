# CLAUDE.md

## 开发环境拓扑（重要）

- **内网编码 + 外网 Figma**：内网无法直连 Figma 和 GitHub；外网访问 GitHub 需代理 `127.0.0.1:7897`（已配置到 git 的 github.com 专属代理）。
- 设计还原工作流：外网机器执行 `/figma-export <figma-url>` 生成自包含还原包 → 传文件到内网 → 内网只读导出包（README.md + tokens.css + metadata.json + assets/ + screenshots/）还原，不连 Figma。
- skill 源码与笔记沉淀在 https://github.com/yuandabo/ai-md
