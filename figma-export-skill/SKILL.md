# figma-export — 一键导出 Figma 设计还原包

把指定 Figma 节点（页面/Frame/组件）的全部还原所需信息导出成一个**自包含目录**，供内网环境（无 Figma 访问）高保真还原使用。

## 触发方式

用户说 `/figma-export <figma-url> [输出目录]` 或「导出这个 Figma 设计」。

- `<figma-url>`：Figma 文件或节点链接（可带 `?node-id=`）
- `[输出目录]`：默认 `./figma-export-<文件名>-<日期>/`

## 前置条件

1. **必须先加载 `figma:figma-use` skill**（Plugin API 规则），并遵循其全部规则（return ID、字体加载、原子性等）
2. **必须先调用 `create_session`**（fileKey 已知后），所有文件工具都要带返回的 sessionId
3. 从 URL 解析 fileKey / nodeId（`node-id=12-345` → `12:345`）

## 导出流程（严格按顺序，每步验证后再下一步）

### Step 0: 建目录

在输出目录下创建结构：

```
<outputDir>/
├── tokens/tokens.json
├── tokens/tokens.css
├── assets/
├── screenshots/
├── metadata.json
└── README.md
```

### Step 1: 结构信息 → `metadata.json`

用 `get_metadata` 获取目标节点树；如需更细的字段（fills、字体、圆角、auto-layout 参数），用 `use_figma` 遍历补充：

```js
// use_figma 脚本示例：导出节点树详细数据
const root = await figma.getNodeByIdAsync("<NODE_ID>");
function walk(n, depth) {
  const item = {
    id: n.id, name: n.name, type: n.type,
    x: n.x, y: n.y, w: n.width, h: n.height,
  };
  if ("fills" in n && Array.isArray(n.fills) && n.fills.length)
    item.fills = n.fills.filter(f => f.visible !== false);
  if (n.type === "TEXT") {
    item.characters = n.characters;
    item.fontName = n.fontName; item.fontSize = n.fontSize;
    item.lineHeight = n.lineHeight; item.letterSpacing = n.letterSpacing;
  }
  if ("cornerRadius" in n && typeof n.cornerRadius === "number") item.cornerRadius = n.cornerRadius;
  if ("layoutMode" in n && n.layoutMode !== "NONE") {
    item.layoutMode = n.layoutMode;
    item.itemSpacing = n.itemSpacing;
    item.paddingLeft = n.paddingLeft; item.paddingTop = n.paddingTop;
    item.paddingRight = n.paddingRight; item.paddingBottom = n.paddingBottom;
  }
  if (depth < 12 && "children" in n) item.children = n.children.map(c => walk(c, depth + 1));
  return item;
}
return walk(root, 0);
```

返回结果原样写入 `metadata.json`。**注意分批**：节点数 >500 时按子树分多次导出再合并，单次调用不超过 10 个逻辑操作的限制不适用于纯读取，但过大返回值会截断——按 Frame 逐个导。

### Step 2: 变量/Tokens → `tokens.json` + `tokens.css`

用 `get_variable_defs` 获取变量；或 `use_figma`：

```js
const collections = await figma.variables.getLocalVariableCollectionsAsync();
const out = [];
for (const col of collections) {
  for (const vid of col.variableIds) {
    const v = await figma.variables.getVariableByIdAsync(vid);
    out.push({
      collection: col.name, mode: col.modes[0].name,
      name: v.name, type: v.type,
      scopes: v.scopes,
      value: v.valuesByMode[col.modes[0].modeId], // RGBA 0-1
    });
  }
}
return out;
```

写入 `tokens.json`，同时生成 `tokens.css`（转换规则见下）。若文件使用的是**库变量**（非本地），用 `get_libraries` / `search_design_system` 获取。

### Step 3: 截图 → `screenshots/`

用 `get_screenshot` 或 `use_figma` 内 `await node.screenshot({scale: 2})`：
- 整页 1 张全景图（scale 自适应，长边 ≤2048px）
- 每个**一级子 Frame** 各 1 张（scale 2）
- 截图命名：`<序号>-<节点名slug>.png`

### Step 4: 素材导出 → `assets/`

对目标子树中的 IMAGE 填充节点和矢量图标，用 `download_assets`（或 `use_figma` 读 `imageBytes`/`exportAsync`）批量导出：
- 图标/矢量 → SVG（`node.exportAsync({format: 'SVG'})`，base64 落盘）
- 位图 → PNG 2x
- 命名用节点名，冲突时追加节点 ID 后 4 位

```js
// use_figma 批量导出 SVG 示例（分批，每批 ≤50 个）
const nodes = root.findAllWithCriteria({ types: ['VECTOR', 'BOOLEAN_OPERATION'] });
return nodes.slice(0, 50).map(n => ({
  id: n.id, name: n.name,
  svg: Buffer.from(n.exportAsync({ format: 'SVG' })).toString('base64'),
}));
```

### Step 5: 生成 `README.md`

内容必须包含：
- 来源 URL、fileKey、nodeId、导出时间
- 页面/Frame 清单及对应截图文件名
- tokens 使用说明（哪个 collection 对应 CSS 前缀）
- 素材清单及用途备注
- 还原注意事项（如裁切文本、绝对定位区域、字体清单——从 metadata 中统计 `fontName` 去重生成）

## Token → CSS 转换规则

- `color` 类型：`{r,g,b}` 0-1 → `#RRGGBB`（有 `a` 则 `rgba()`）；变量名 `color/bg/primary` → `--color-bg-primary`（`/` 分隔的路径转 `-`）
- `float` 尺寸类（scopes 含 GAP/PADDING 等）→ `--spacing-xxx: <px>`
- `string`（FONT_FAMILY 等）→ 原样字符串
- 文件头注明来源 collection 和 mode

## 关键纪律

- 每步完成后向用户报告进度（发现 N 个节点/导出 M 张图）
- 单个 `use_figma` 返回值过大（>100KB）时主动分批
- 所有落盘文件用 Write/Bash 工具写本地，不依赖 MCP
- 导出完成后校验：README 里的清单与实际文件一一对应，报告最终文件树和总体积
- 任何一步失败：按 figma-use 的错误恢复规则处理，不跳过
