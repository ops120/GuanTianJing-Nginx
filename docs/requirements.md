# Mirror Nginx - Nginx 配置文件拓扑分析工具

## 项目概述

Mirror Nginx 是一款专业的 Nginx 配置文件拓扑分析工具，支持多文件解析、链路关系提取和可视化拓扑图生成。

### 核心功能

1. **多文件配置解析**
   - 支持批量上传多个 Nginx 配置文件
   - 自动识别 server、upstream、location 块
   - 提取 proxy_pass 转发规则

2. **链路关系分析**
   - 识别 Nginx 之间的链路转发关系
   - 追踪 upstream 到后端服务器的映射
   - 构建完整的拓扑流向图

3. **可视化拓扑图**
   - 交互式拓扑图展示
   - 支持节点拖动调整位置
   - 支持鼠标滚轮缩放
   - 支持画布拖动平移
   - 一键恢复初始视图

4. **实时解析预览**
   - 实时显示解析统计信息
   - 展示所有提取的 server、upstream、location
   - 显示链路关系列表

## 功能需求

### 1. 文件上传功能

**描述**: 支持用户上传多个 Nginx 配置文件进行分析

**功能点**:
- 支持拖拽上传和点击上传
- 支持 .conf、.txt、.nginx 文件格式
- 支持多文件批量上传
- 显示已上传文件列表
- 支持删除已上传文件
- 支持编辑配置文件内容
- 自动触发拓扑生成

### 2. 配置解析功能

**描述**: 解析 Nginx 配置文件，提取关键信息

**解析内容**:
- **Server 块**: 提取 listen 端口、server_name
- **Upstream 块**: 提取 upstream 名称及后端服务器列表
- **Location 块**: 提取路由路径和配置类型
- **Proxy Pass**: 识别转发目标和类型（upstream 或直接 IP）

**链路类型识别**:
- `nginx_chain`: Nginx 之间的直接转发（server → IP）
- `server_to_upstream`: Server 转发到 Upstream
- `upstream_backend`: Upstream 到后端服务器

### 3. 拓扑图生成

**描述**: 根据解析结果生成可视化拓扑图

**拓扑布局**:
- Layer 0: Client（客户端）
- Layer 1: 第一层 Server 节点
- Layer 2: 第二层 Server 节点 + Upstream 节点
- Layer 3: Backend 节点

**节点类型**:
- **Client**: 客户端节点（灰色）
- **Server**: Nginx Server 节点（紫色）
- **Upstream**: Upstream 服务池节点（青色）
- **Backend**: 后端服务器节点（黄色）

**连线类型**:
- **Internal**: Client 到 Server（绿色渐变）
- **Proxy**: Server 之间转发（紫红渐变）
- **Upstream**: Server 到 Upstream、Upstream 到 Backend（青色渐变）

### 4. 交互功能

**拖动节点**:
- 每个节点都可以独立拖动
- 拖动时节点半透明显示
- 悬停时节点发光高亮
- 连线实时跟随更新

**缩放功能**:
- 鼠标滚轮缩放（以鼠标位置为中心）
- 缩放范围: 0.3x - 3x
- 支持按钮放大/缩小

**平移功能**:
- 拖动画布空白区域进行平移
- 拖动时光标变为 grabbing

**一键复原**:
- 重置视图到初始状态
- 清除所有节点拖动调整

### 5. 分析面板

**描述**: 实时显示解析结果统计

**显示内容**:
- 配置文件数量
- Server 块数量
- Location 路由数量
- Upstream 数量
- Proxy 转发数量
- 链路关系列表

## 技术栈

- **前端框架**: 纯 HTML/CSS/JavaScript
- **图表库**: SVG 原生渲染
- **样式**: CSS3 动画和过渡效果
- **字体**: JetBrains Mono + Space Grotesk

## 配置文件示例

### 示例 1: a.conf
```nginx
server {
    listen 80;
    server_name www.123.com;

    location / {
        proxy_pass http://192.168.1.11;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 示例 2: b.conf
```nginx
server {
    listen 80;
    server_name 192.168.1.11;

    location / {
        proxy_pass http://192.168.1.12;
        proxy_set_header Host $host;
    }
}
```

### 示例 3: c.conf
```nginx
upstream backend_servers {
    server 127.0.0.1:8080;
    server 192.168.2.123:9090;
}

server {
    listen 80;
    server_name 192.168.1.12;

    location / {
        proxy_pass http://backend_servers;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }
}
```

### 预期链路关系

```
Client
  ↓
www.123.com:80 (a.conf)
  ↓ 192.168.1.11
192.168.1.11:80 (b.conf)
  ↓ 192.168.1.12
192.168.1.12:80 (c.conf)
  ↓ backend_servers
backend_servers
  ↓
127.0.0.1:8080  192.168.2.123:9090
```

## 用户界面布局

```
┌─────────────────────────────────────────────────────────┐
│  Header: Logo + 统计信息                               │
├────────────┬──────────────────────────┬────────────────┤
│            │                          │                │
│  文件上传   │                          │   解析结果      │
│  文件列表   │     拓扑图画布           │   统计面板      │
│  快速示例   │                          │   链路关系      │
│            │                          │                │
│            │     [控制按钮]           │                │
│            │                          │                │
├────────────┴──────────────────────────┴────────────────┤
│  图例说明                                                │
└─────────────────────────────────────────────────────────┘
```

## 性能要求

- 文件上传到解析完成时间 < 1秒
- 拓扑图渲染时间 < 500ms
- 节点拖动响应时间 < 16ms（60fps）
- 支持最多 100 个节点同时显示

## 兼容性要求

- 支持现代浏览器（Chrome、Firefox、Edge、Safari）
- 最小支持分辨率: 1280x720
- 响应式设计，适配不同屏幕尺寸

## 未来扩展方向

1. 导出拓扑图为图片（PNG/SVG）
2. 导入真实网络设备配置
3. 支持更多负载均衡算法可视化
4. 支持集群模式分析
5. 添加节点信息编辑功能
6. 支持配置文件语法检查
7. 生成配置变更建议

## 成功标准

1. ✅ 能够正确解析所有标准 Nginx 配置文件
2. ✅ 能够准确识别链路关系类型
3. ✅ 拓扑图能够清晰展示转发流程
4. ✅ 交互操作流畅无卡顿
5. ✅ 界面美观，符合现代设计趋势
