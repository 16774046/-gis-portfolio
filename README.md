# 个人 GIS 作品集 — 北京城市 POI 综合地图

## 第 12 讲 P1 综合实验报告

---

## 一、项目概述

本项目基于 **Leaflet 1.9.4** 与 **MapLibre GL JS 4.7+**，构建了一个功能完备的 Web GIS 应用——北京城市 POI（兴趣点）综合地图。项目整合了第 08-11 讲所学内容，实现了多底图切换、多类型 POI 标注与筛选、地图书签与视角定位、比例尺/坐标显示/距离测量等核心功能，并支持 2D 地图与 3D 地形的无缝切换。

**在线访问：** [https://YOUR_USERNAME.github.io/gis-portfolio/](https://YOUR_USERNAME.github.io/gis-portfolio/)（部署后替换为实际链接）

---

## 二、功能清单

| 功能模块 | 对应课程 | 实现内容 |
|---------|---------|---------|
| 多底图加载与切换 | 第 08 讲练习 2 | 5 种底图（OSM / 天地图矢量 / 天地图卫星 / CartoDB 浅色 / CartoDB 暗色），Leaflet Layer Control 切换 |
| 多类型 POI 标注 | 第 09 讲练习 1 | 5 大类 20 个 POI，自定义 DivIcon 彩色图标，Popup 详情弹窗，Tooltip 悬浮提示 |
| 地图书签/视角定位 | 第 08 讲挑战 1 | 8 个预设书签，点击飞行定位，键盘快捷键 (1-4,0) 快速切换 |
| 比例尺 & 坐标显示 | 第 09 讲练习 2 | Leaflet 内置比例尺，鼠标实时 WGS84 坐标显示（6 位小数） |
| 距离测量工具 | 第 09 讲练习 2 | 点击打点→连线段→Haversine 公式计算距离→标签显示段长与总长 |
| 3D 地形视图 | 综合拓展 | MapLibre GL JS 渲染全球地形，Pitch/Roll 三维视角，大气雾效 |
| POI 分类筛选 | 综合拓展 | 按类别独立开关过滤 POI 显示 |
| 分层图例 | 综合拓展 | 颜色编码的类别图例说明 |

---

## 三、技术架构

### 3.1 技术栈

```
┌─────────────────────────────────────────────┐
│                  index.html                  │
│  ┌─────────────┐  ┌──────────────────────┐  │
│  │   Sidebar   │  │     Map Container     │  │
│  │  · Bookmarks│  │  ┌─────────────────┐  │  │
│  │  · Filters  │  │  │  Leaflet Map    │  │  │
│  │  · Measure  │  │  │  (2D Primary)   │  │  │
│  │  · Coords   │  │  └─────────────────┘  │  │
│  │  · Legend   │  │  ┌─────────────────┐  │  │
│  └─────────────┘  │  │ MapLibre GL Map │  │  │
│                    │  │ (3D Terrain)    │  │  │
│                    │  └─────────────────┘  │  │
│                    └──────────────────────┘  │
└─────────────────────────────────────────────┘
```

- **Leaflet 1.9.4**：2D 主地图引擎，负责底图管理、POI 标注、图层控制、测量交互
- **MapLibre GL JS 4.7.1**：3D 地形渲染引擎，支持 WebGL 地形、天空盒、大气雾效
- **GeoJSON**：POI 数据交换格式，通过 Fetch API 异步加载

### 3.2 数据流

```
beijing-pois.geojson ──Fetch──> allPois[] ──Filter──> renderPOIs()
                                                      │
                                          ┌───────────┘
                                          ▼
                                    poiLayerGroup
                                    (Leaflet Layer)
                                          │
                              ┌───────────┼───────────┐
                              ▼           ▼           ▼
                          DivIcon      Tooltip      Popup
                       (color+emoji)  (hover)     (click)
```

### 3.3 底图图层结构

```
Layer Control
├── OpenStreetMap 标准      (L.tileLayer)
├── 天地图 矢量             (L.layerGroup: vec_w + cva_w)
├── 天地图 卫星             (L.layerGroup: img_w + cia_w)
├── CartoDB 浅色           (L.tileLayer)
└── CartoDB 暗色           (L.tileLayer)
```

---

## 四、核心实现

### 4.1 自定义 POI 图标（DivIcon）

```javascript
function makeIcon(category, isSelected) {
  const color = CAT_COLORS[category];
  const emoji = CAT_ICONS[category];
  // 旋转 45° 的圆角方形 + 内部反向旋转的 emoji
  return L.divIcon({
    html: `<div style="...transform:rotate(-45deg);border-radius:50% 50% 50% 0">
             <span style="...transform:rotate(45deg)">${emoji}</span>
           </div>`,
    iconSize: [32, 32],
    iconAnchor: [16, 32],
    popupAnchor: [0, -32]
  });
}
```

设计思路：使用 CSS transform 实现菱形标记（对角线方向），内部 emoji 反向旋转保持正立。按类别着色提升视觉区分度。

### 4.2 Haversine 距离测量

```javascript
function haversine(lat1, lon1, lat2, lon2) {
  const R = 6371000; // 地球半径（米）
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLon = (lon2 - lon1) * Math.PI / 180;
  const a = Math.sin(dLat/2)**2 +
            Math.cos(lat1*Math.PI/180) * Math.cos(lat2*Math.PI/180) *
            Math.sin(dLon/2)**2;
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
}
```

测距交互流程：点击"测距"按钮 → 地图 cursor 变为十字 → 每次点击添加测点并连线 → 线段中点显示距离 → 双击结束测量 → 显示总距离。

### 4.3 地图书签飞行定位

使用 Leaflet `flyTo()` 方法实现平滑的相机飞行动画（duration: 1.5s），支持键盘数字键快捷定位：
- `1` - 天安门广场 | `2` - 故宫 | `3` - 颐和园 | `4` - 八达岭长城 | `0` - 北京全景

### 4.4 2D/3D 视图切换

视图切换时自动同步地图中心与缩放级别：
```javascript
const lCenter = leafletMap.getCenter();
const lZoom   = leafletMap.getZoom();
maplibreMap.jumpTo({
  center: [lCenter.lng, lCenter.lat],
  zoom: lZoom - 0.5
});
```

### 4.5 MapLibre 3D 地形

使用 AWS Terrarium 全球 DEM 瓦片作为高程数据源，结合 MapLibre 的天空盒与大气雾效渲染真实 3D 地形场景。

---

## 五、数据说明

### 5.1 POI 数据

| 文件 | 格式 | 记录数 | 字段 |
|------|------|--------|------|
| `data/beijing-pois.geojson` | GeoJSON FeatureCollection | 20 | name, category, address, description, rating, hours |

### 5.2 分类统计

| 类别 | 颜色 | 数量 | 代表 POI |
|------|------|------|---------|
| 景点名胜 | 🟠 #E67E22 | 8 | 天安门、故宫、颐和园、八达岭长城、天坛、鸟巢、水立方、南锣鼓巷 |
| 交通枢纽 | 🔵 #3498DB | 3 | 首都机场 T3、北京站、北京南站 |
| 餐饮美食 | 🔴 #E74C3C | 2 | 王府井小吃街、簋街 |
| 购物商圈 | 🟢 #27AE60 | 3 | 三里屯太古里、国贸商城、西单大悦城 |
| 文化教育 | 🟣 #9B59B6 | 4 | 清华大学、北京大学、798 艺术区、国家大剧院 |

### 5.3 底图数据源

- **OpenStreetMap**: 开源全球地图数据
- **天地图 (Tianditu)**: 国家地理信息公共服务平台，需申请 API Key
- **CartoDB**: 基于 OSM 数据的定制化底图样式

---

## 六、部署指南

### 方式一：GitHub Pages（推荐）

```bash
# 1. 创建仓库并推送代码
cd gis-portfolio
git init
git add -A
git commit -m "Initial commit: GIS portfolio project"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/gis-portfolio.git
git push -u origin main

# 2. 在 GitHub 仓库 Settings > Pages 中启用
#    选择 Source: Deploy from a branch, Branch: main, / (root)
```

### 方式二：本地运行

```bash
cd gis-portfolio
# Python 3
python -m http.server 8080
# 或 Node.js
npx serve .
# 访问 http://localhost:8080
```

### 方式三：任意静态服务器

将项目目录完整上传至任意静态 Web 服务器（Nginx / Apache / OSS / CloudFront 等），确保 `data/` 目录可被访问。

---

## 七、天地图 Key 配置

默认使用公开 Demo Key。如需自己的 Key：

1. 注册天地图开发者：https://console.tianditu.gov.cn/
2. 创建应用获取 Key (tk)
3. 修改 `index.html` 顶部的 `TIANDITU_KEY` 变量

---

## 八、浏览器兼容性

| 浏览器 | 2D (Leaflet) | 3D (MapLibre GL) |
|--------|:-----------:|:-----------------:|
| Chrome 90+ | ✅ | ✅ |
| Edge 90+ | ✅ | ✅ |
| Firefox 90+ | ✅ | ✅ |
| Safari 15+ | ✅ | ✅ |

MapLibre GL JS 需要 WebGL 支持。

---

## 九、项目结构

```
gis-portfolio/
├── index.html                # 主页面（HTML + CSS + JS）
├── data/
│   └── beijing-pois.geojson  # 北京 POI 空间数据
└── README.md                 # 实验报告（本文件）
```

---

## 十、实验总结

本项目综合运用了 Web GIS 开发的核心技术：

1. **底图加载**：掌握了 WMTS 瓦片服务（天地图）与标准 TMS 服务（OSM/CartoDB）的 Leaflet 接入方法，理解了 Layer Control 的实现原理
2. **POI 标注**：使用 GeoJSON 结构化存储数据，通过 L.divIcon 实现自定义图标，Popup/Tooltip 提供交互信息展示
3. **空间测量**：实现了基于 Haversine 公式的大圆距离计算，完成了交互式测距工具
4. **3D 可视化**：引入 MapLibre GL JS，配置地形 DEM 数据源与大气雾效，实现 2D/3D 联动切换
5. **工程实践**：采用 Fetch API 异步数据加载 + 本地回退兜底策略，确保离线可用性；响应式布局适配桌面与移动端

**待改进方向：** 接入更多矢量瓦片数据源；添加热力图/聚类可视化；实现书签的增删改与浏览器 localStorage 持久化；添加面积测量功能。

---

*实验日期：2026 年 6 月*
