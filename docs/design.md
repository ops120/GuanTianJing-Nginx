# Mirror Nginx - 技术设计文档

## 系统架构

### 整体架构

```
┌─────────────────────────────────────────┐
│           用户界面层 (UI Layer)           │
│  ┌─────────┬──────────┬──────────┐    │
│  │ 文件上传 │  拓扑画布  │  分析面板 │    │
│  └─────────┴──────────┴──────────┘    │
├─────────────────────────────────────────┤
│          业务逻辑层 (Business Logic)      │
│  ┌─────────────┬──────────────────┐    │
│  │  文件管理器  │   拓扑生成器    │    │
│  │  解析引擎   │   渲染引擎      │    │
│  └─────────────┴──────────────────┘    │
├─────────────────────────────────────────┤
│            数据模型层 (Data Model)        │
│  ┌─────────┬─────────┬──────────┐      │
│  │  节点   │   边    │  链路关系 │      │
│  └─────────┴─────────┴──────────┘      │
└─────────────────────────────────────────┘
```

### 技术选型

| 层级 | 技术 | 说明 |
|------|------|------|
| UI | HTML5 + CSS3 | 语义化标签 + 现代CSS特性 |
| 图形渲染 | SVG | 矢量图形，支持缩放不失真 |
| 交互 | Vanilla JavaScript | 原生JS，无框架依赖 |
| 样式 | CSS Variables + Flexbox | 响应式布局 + 主题系统 |
| 字体 | JetBrains Mono + Space Grotesk | 等宽代码 + 现代UI字体 |

## 核心模块设计

### 1. 文件管理器 (FileManager)

**职责**: 处理文件上传、存储和访问

**核心功能**:
```javascript
class FileManager {
    uploadedFiles: Array<FileData>
    selectedFileId: string | null
    
    handleFiles(files: FileList): void
    selectFile(id: string): void
    deleteFile(id: string): void
    updateFileContent(id: string, content: string): void
}
```

**数据结构**:
```javascript
interface FileData {
    id: string          // 唯一标识
    name: string        // 文件名
    content: string     // 文件内容
}
```

### 2. 配置解析器 (ConfigParser)

**职责**: 解析 Nginx 配置文件，提取关键信息

**解析策略**:
1. 使用正则表达式提取各类型块
2. 识别链路关系类型
3. 构建解析结果数据

**核心正则表达式**:
```javascript
// Upstream 块解析
/upstream\s+([\w_]+)(?::(\d+))?\s*\{([^}]*)\}/g

// Server 块解析
/server\s*\{([\s\S]*?)(?=\n\s*server\s*\{|\n\s*upstream\s*|$)/g

// Location 块解析
/location\s+([^\s{]+)\s*\{([\s\S]*?)\n\s*\}/g

// Proxy Pass 解析
/proxy_pass\s+([^;]+);/

// IP 地址提取
/https?:\/\/(\d+\.\d+\.\d+\.\d+):?(\d+)?/
```

**解析结果结构**:
```javascript
interface ParsedConfig {
    upstreams: Array<Upstream>
    servers: Array<Server>
    locations: Array<Location>
    proxies: Array<Proxy>
    links: Array<Link>
}

interface Link {
    type: 'nginx_chain' | 'server_to_upstream' | 'upstream_backend'
    source: string
    target: string
    port?: string
    location?: string
    sourceFile: string
}
```

### 3. 拓扑生成器 (TopologyGenerator)

**职责**: 根据链路关系构建拓扑数据结构

**布局算法**:
```javascript
function renderTopologyFromLinks(centerX: number) {
    // 1. 创建节点映射
    const nodeMap = new Map<string, Node>()
    
    // 2. 分层布局
    // Layer 0: Client (y=60)
    // Layer 1: 第一层 Servers (y=150)
    // Layer 2: 第二层 Servers + Upstreams (y=280)
    // Layer 3: Backends (y=410)
    
    // 3. 节点位置计算
    nodeMap.forEach(node => {
        node.x = centerX - totalWidth/2 + index * spacing
        node.y = layerY
    })
    
    // 4. 构建边
    const edges = links.map(link => createEdge(link))
}
```

**节点类型**:
```javascript
enum NodeType {
    CLIENT = 'client'      // 灰色
    SERVER = 'server'      // 紫色
    UPSTREAM = 'upstream'  // 青色
    BACKEND = 'backend'    // 黄色
}
```

### 4. SVG 渲染引擎 (Renderer)

**职责**: 将拓扑数据渲染为可交互的 SVG 图形

**渲染流程**:
```javascript
function renderSVG(skipViewBoxUpdate = false) {
    // 1. 计算 viewBox
    if (!skipViewBoxUpdate) {
        calculateViewBox()
    }
    
    // 2. 渲染边（线条）
    const edgesHtml = topologyEdges.map(renderEdge)
    
    // 3. 渲染节点（圆形 + 图标）
    const nodesHtml = topologyNodes.map(renderNode)
    
    // 4. 组合输出
    svg.innerHTML = gradientDefs + edgesHtml + nodesHtml
}
```

**SVG 图形元素**:
```html
<svg viewBox="0 0 1000 700">
    <defs>
        <!-- 渐变定义 -->
        <linearGradient id="edgeCyan">...</linearGradient>
        <marker id="arrowCyan">...</marker>
    </defs>
    
    <!-- 边 -->
    <path d="M500,60 L500,150" stroke="url(#edgeGreen)" />
    
    <!-- 节点组 -->
    <g transform="translate(500, 60)">
        <circle r="25" fill="#1a1a25" stroke="#ffffff" />
        <text>Client</text>
    </g>
</svg>
```

### 5. 交互控制器 (InteractionController)

**职责**: 处理用户交互操作

**交互类型**:

#### 5.1 节点拖动
```javascript
// 拖动状态
let isDraggingNode = false
let draggedNodeId = null
let dragStartPos = { x: 0, y: 0 }
let dragNodeStartPos = { x: 0, y: 0 }

// 开始拖动
function startNodeDrag(event, nodeId) {
    isDraggingNode = true
    draggedNodeId = nodeId
    dragStartPos = { x: event.clientX, y: event.clientY }
    dragNodeStartPos = { x: node.x, y: node.y }
}

// 拖动中
function onNodeDrag(event) {
    const dx = event.clientX - dragStartPos.x
    const dy = event.clientY - dragStartPos.y
    
    const scaleX = viewBoxState.width / canvasWidth
    const scaleY = viewBoxState.height / canvasHeight
    
    node.x = dragNodeStartPos.x + dx * scaleX
    node.y = dragNodeStartPos.y + dy * scaleY
    
    // 跳过 viewBox 更新，防止缩放变化
    renderSVG(true)
}

// 结束拖动
function stopNodeDrag() {
    isDraggingNode = false
    draggedNodeId = null
}
```

#### 5.2 画布平移
```javascript
// 平移状态
let isPanning = false
let startPan = { x: 0, y: 0 }

// 鼠标按下开始平移
canvas.addEventListener('mousedown', (e) => {
    if (e.button === 0) {
        isPanning = true
        startPan = { x: e.clientX, y: e.clientY }
    }
})

// 鼠标移动平移
window.addEventListener('mousemove', (e) => {
    if (!isPanning) return
    
    const dx = e.clientX - startPan.x
    const dy = e.clientY - startPan.y
    
    const scaleX = viewBoxState.width / canvasWidth
    const scaleY = viewBoxState.height / canvasHeight
    
    viewBoxState.x -= dx * scaleX
    viewBoxState.y -= dy * scaleY
    
    applyPanAndZoom()
})
```

#### 5.3 滚轮缩放
```javascript
canvas.addEventListener('wheel', (e) => {
    e.preventDefault()
    
    const delta = e.deltaY > 0 ? 1.1 : 0.9
    
    // 计算鼠标在 SVG 中的位置
    const rect = canvas.getBoundingClientRect()
    const mouseX = e.clientX - rect.left
    const mouseY = e.clientY - rect.top
    
    const svgPointX = viewBoxState.x + (mouseX/rect.width) * viewBoxState.width
    const svgPointY = viewBoxState.y + (mouseY/rect.height) * viewBoxState.height
    
    // 调整 viewBox 以鼠标位置为中心缩放
    const newWidth = viewBoxState.width * delta
    const newHeight = viewBoxState.height * delta
    
    viewBoxState.x = svgPointX - (mouseX/rect.width) * newWidth
    viewBoxState.y = svgPointY - (mouseY/rect.height) * newHeight
    viewBoxState.width = newWidth
    viewBoxState.height = newHeight
    
    applyPanAndZoom()
}, { passive: false })
```

## 数据流设计

### 完整数据流

```
用户上传文件
    ↓
FileManager.handleFiles()
    ↓
读取文件内容 → stored in uploadedFiles[]
    ↓
自动触发 updateStats()
    ↓
ConfigParser.parseNginxConfig()
    ↓
生成 ParsedConfig { upstreams, servers, locations, proxies, links }
    ↓
自动触发 generateTopology()
    ↓
TopologyGenerator.renderTopologyFromLinks()
    ↓
生成 topologyNodes[] 和 topologyEdges[]
    ↓
Renderer.renderSVG()
    ↓
SVG DOM 更新 → 拓扑图显示
```

### 状态管理

```javascript
// 全局状态
let uploadedFiles = []           // 上传的文件列表
let selectedFileId = null        // 当前选中的文件
let parsedData = {              // 解析结果
    upstreams: [],
    servers: [],
    locations: [],
    proxies: [],
    links: []
}
let topologyNodes = []           // 拓扑节点
let topologyEdges = []          // 拓扑边
let viewBoxState = {            // 视图状态
    x: 0,
    y: 0,
    width: 1000,
    height: 700
}
let initialViewBoxState = {...}  // 初始视图状态
let isDraggingNode = false      // 节点拖动状态
let isPanning = false           // 画布平移状态
```

## UI 组件设计

### 1. 拓扑容器

```css
.topology-container {
    position: relative;
    background: var(--bg-primary);
    border: 1px solid var(--border-color);
    border-radius: 12px;
    overflow: hidden;
}

.topology-svg {
    width: 100%;
    height: 100%;
}
```

### 2. 节点样式

```css
.topology-node {
    cursor: move;
    transition: opacity 0.2s;
}

.topology-node:hover circle {
    stroke-width: 3;
    filter: drop-shadow(0 0 8px currentColor);
}

.topology-node.dragging {
    opacity: 0.8;
}
```

### 3. 控制按钮

```css
.control-btn {
    width: 32px;
    height: 32px;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    border-radius: 6px;
    color: var(--text-secondary);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s;
}

.control-btn:hover {
    background: var(--bg-tertiary);
    border-color: var(--accent-cyan);
    color: var(--accent-cyan);
}
```

## 性能优化策略

### 1. SVG 渲染优化

```javascript
// 使用 requestAnimationFrame 节流
let rafId = null
function onNodeDrag(event) {
    if (rafId) return
    
    rafId = requestAnimationFrame(() => {
        updateNodePosition(event)
        renderSVG(true)
        rafId = null
    })
}
```

### 2. 事件委托

```javascript
// 在 SVG 容器上委托事件，而不是每个节点
svg.addEventListener('mousedown', (e) => {
    const nodeElement = e.target.closest('.topology-node')
    if (nodeElement) {
        const nodeId = extractNodeId(nodeElement)
        startNodeDrag(e, nodeId)
    } else {
        startCanvasPan(e)
    }
})
```

### 3. 避免不必要的重渲染

```javascript
// 添加标志位跳过 viewBox 更新
function renderSVG(skipViewBoxUpdate = false) {
    if (!skipViewBoxUpdate) {
        calculateViewBox()  // 昂贵的计算
    }
    renderNodes()
    renderEdges()
}
```

## 文件结构

```
MirrorNet/
├── index.html              # 主页面（包含所有功能）
├── docs/
│   ├── requirements.md     # 需求文档
│   └── design.md          # 设计文档（本文档）
├── example/
│   ├── a.conf            # 示例配置文件 1
│   ├── b.conf            # 示例配置文件 2
│   └── c.conf            # 示例配置文件 3
└── ...
```

## 浏览器兼容性

| 特性 | Chrome | Firefox | Safari | Edge |
|------|--------|---------|--------|------|
| SVG | ✅ | ✅ | ✅ | ✅ |
| CSS Variables | ✅ | ✅ | ✅ | ✅ |
| Flexbox | ✅ | ✅ | ✅ | ✅ |
| requestAnimationFrame | ✅ | ✅ | ✅ | ✅ |
| Wheel Event | ✅ | ✅ | ✅ | ✅ |

## 扩展性设计

### 模块化架构

```javascript
// 便于扩展的模块接口
class Parser {
    parse(config: string): ParsedConfig
}

class TopologyBuilder {
    build(parsedConfig: ParsedConfig): TopologyData
}

class Renderer {
    render(topologyData: TopologyData): SVGElement
}

class InteractionManager {
    enableDrag(node: Node): void
    enableZoom(): void
    enablePan(): void
}
```

### 插件化支持

未来可以支持:
1. 自定义解析器（支持 Apache、Nginx+、HAProxy）
2. 自定义渲染器（支持 Canvas、WebGL）
3. 自定义布局算法（力导向、树状图）

## 测试策略

### 单元测试
- Parser: 测试各种配置格式解析
- TopologyBuilder: 测试链路关系构建
- Renderer: 测试 SVG 生成

### 集成测试
- 文件上传 → 解析 → 渲染完整流程
- 交互操作响应测试

### 性能测试
- 100+ 节点渲染性能
- 拖动响应时间
- 缩放流畅度

## 部署说明

### 开发环境
```bash
# 直接用浏览器打开 index.html 即可
open index.html
```

### 生产环境
```bash
# 使用任意静态文件服务器
npx serve .
# 或
python -m http.server 8080
```

### 依赖
- 无后端依赖，纯前端应用
- 无外部 JS 库依赖
- 仅依赖 Google Fonts CDN（可选）

## 维护指南

### 添加新节点类型
1. 在 `NodeType` 枚举中添加类型
2. 在 `renderSVG()` 中添加颜色定义
3. 在 `getNodeIcon()` 中添加图标
4. 在 CSS 中添加样式

### 添加新链路类型
1. 在链路解析正则中添加匹配规则
2. 在 `renderTopologyFromLinks()` 中添加处理逻辑
3. 在渲染时添加边的样式

### 国际化支持
1. 提取所有字符串到语言包
2. 使用 `data-i18n` 属性标记
3. 动态加载语言包切换

## 版本历史

### v1.0.0 (当前版本)
- 多文件 Nginx 配置解析
- 链路关系提取
- SVG 拓扑图渲染
- 节点拖动交互
- 画布平移缩放
- 响应式界面设计
