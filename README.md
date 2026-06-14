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
| 分级设色图 (Choropleth) | 第 10 讲练习 1 | 16 区人口密度 6 级 YlOrRd 顺序色带，`style` 回调 + `onEachFeature` Popup/Tooltip/高亮 |
| 比例符号图 (Proportional Symbols) | 第 10 讲练习 2 | GDP 映射圆形半径（√ 变换），`pointToLayer` + `filter` 过滤 + 图例 |
| 专题图切换 | 综合拓展 | 侧边栏 Tab + 右上角 Layer Control 双层控制 |
| 矢量瓦片底图 | 第 11 讲练习 1 | MapLibre Style JSON 加载 OS​M 矢量瓦片，9 层自定义样式（water/landuse/roads/buildings/labels）|
| 数据驱动样式 | 第 11 讲练习 1 | POI GeoJSON Source + 3 层（cluster circles / count labels / individual circles），`match` 表达式按 category 着色 |
| 样式切换器 | 第 11 讲练习 2 | 3 套配色方案（街区/暗色/古风），运行时 `setPaintProperty` 切换 |
| 点聚合可视化 | 第 11 讲挑战 1 | MapLibre 内置 `cluster:true`，4 级圆半径 + 计数标签，点击展开 |
| 建筑 3D 拉伸 | 综合拓展 | `fill-extrusion` 图层，`coalesce(height, render_height, levels*3)` 提取高度，按高度渐变色 |

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

### 4.0 分层设色图配色方案

**采用 YlOrRd (Yellow-Orange-Red) 6 级顺序色带：**

```
密度 (人/km²)    颜色        色值
─────────────────────────────────
   0 –   300   浅黄        #ffffb2
 300 – 1,000   淡黄        #fed976
1000 – 3,000   浅橙        #feb24c
3000 – 8,000   中橙        #fd8d3c
8000 –15,000   深橙红      #f03b20
15000–25,000   暗红        #bd0026
   > 25,000   深暗红      #800026
```

**选择依据（顺序色带 vs 发散色带 vs 定性色带）：**

| 色带类型 | 适用场景 | 本例是否适用 |
|---------|---------|:----------:|
| **顺序 (Sequential)** | 数值从低到高连续变化，如密度、比率、计数 | ✅ 选中 |
| 发散 (Diverging) | 数值有双向偏离中心值，如±变化率、收支差 | ✗ 无负值/无临界中点 |
| 定性 (Qualitative) | 分类数据，无大小顺序，如土地类型 | ✗ 密度有明确的低-高顺序 |

**具体理由：**
1. 人口密度是**单向递增**的连续变量（300 → 25,000+ 人/km²），不存在"零中点"或"正向/负向"分化的语义结构，因此排除发散色带。
2. **单一色相渐变**（黄→橙→红）比多色相渐变更能保持读者对"低→高"的**感知保序性**——颜色越深越"重"，直觉对应更高密度。
3. YlOrRd 在 ColorBrewer 中为 colorblind-safe 的 6 级方案，可读性经过验证。
4. 跨类别数据（如城区 vs 郊区类型）更适合定性色带，但本图映射的是连续统计值，不适用。

**分级方法：** 手动设定 7 档断点（基于 Jenks Natural Breaks 近似），使类内差异最小化、类间差异最大化。考虑到北京城区/郊区密度极端分化（东城 >20,000 vs 延庆 <200），未采用等间距分级，避免低密度区域全挤在浅色带无法区分。

---

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

### 4.6 分级设色图（Choropleth）

**`getColor` 分级函数：**
```javascript
function getColor(density) {
  return density > 25000 ? '#800026' :
         density > 15000 ? '#bd0026' :
         density > 8000  ? '#f03b20' :
         density > 3000  ? '#fd8d3c' :
         density > 1000  ? '#feb24c' :
         density > 300   ? '#fed976' :
                            '#ffffb2';
}
```

**`style` 回调 + `onEachFeature` 交互：**
```javascript
L.geoJSON(allDistricts, {
  style: choroplethStyle,           // 按密度返回 fillColor
  onEachFeature: onEachDistrict     // bindPopup + bindTooltip + hover 高亮
}).addTo(choroplethLayer);
```

每个区的 `onEachFeature` 绑定 3 种交互：
- **Sticky Tooltip** — 鼠标悬浮显示区名 + 密度值
- **Popup** — 点击弹窗显示人口/面积/密度/GDP 完整表格
- **hover 高亮** — 白色边框加粗 + `bringToFront()` 避免被邻近区遮挡

### 4.7 比例符号图（Proportional Symbols）

**`pointToLayer` 回调 — 圆形半径映射 GDP：**
```javascript
function getRadius(gdp) {
  return Math.sqrt(gdp) * 0.85;  // √ 变换补偿面积错觉
}
function pointToLayer(feature, latlng) {
  return L.circleMarker(latlng, {
    radius: getRadius(feature.properties.gdp),
    fillColor: '#e94560', fillOpacity: 0.65
  });
}
```

**`filter` 过滤：**
```javascript
function propSymbolFilter(feature) {
  return feature.properties.gdp > 0; // 所有区通过
}
```

**比例尺度选择的数学依据：** 直接线性映射半径（r ∝ GDP）会导致读者按圆面积比较数值，产生平方级面积错觉（GDP 大 4 倍的区看起来大 16 倍）。采用 `r ∝ √GDP` 使面积正比于 GDP，符合 Weber-Fechner 视觉感知规律。

### 4.8 专题图切换机制

两层控制互不冲突：
- **侧边栏 Tab**（分级设色 / 比例符号 / 关闭）— 控制当前显示哪个专题图层
- **右上角 Layer Control** — 勾选/取消专题图层（与底图切换共用菜单）

### 4.9 MapLibre 矢量瓦片底图（Style JSON）

使用 OpenFreeMap 提供的免费 OSM 矢量瓦片 (`tiles.openfreemap.org/planet/{z}/{x}/{y}.mvt`)，
通过 MapLibre Style JSON 构建完整的自定义底图：

```javascript
sources: {
  'osm-vector': {
    type: 'vector',
    tiles: ['https://tiles.openfreemap.org/planet/{z}/{x}/{y}.mvt'],
    maxzoom: 14
  }
}
```

图层定义（9 层）：
| 图层 ID | 类型 | 数据源层 | 说明 |
|---------|------|---------|------|
| background | background | — | 底色 |
| landuse-residential | fill | landuse | 住宅用地 |
| landuse-park | fill | landuse | 公园绿地 |
| landuse-hospital | fill | landuse | 医院 |
| landuse-school | fill | landuse | 学校 |
| water | fill | water | 水域 |
| water-outline | line | water | 水域边线 |
| road-major | line | transportation | 主干道 |
| road-minor | line | transportation | 支路 |
| buildings-3d | fill-extrusion | building | 3D 建筑拉伸 |
| place-labels | symbol | place | 地名标注 |
| road-labels | symbol | transportation_name | 道路名标注 |

### 4.10 GeoJSON Source/Layer + 数据驱动样式

POI 数据通过 `maplibreMap.addSource('beijing-pois', { type:'geojson', data: fc })` 注入。

**3 层渲染架构：**
```
beijing-pois (GeoJSON Source)
├── poi-clusters        → type:circle   | filter: [has, point_count]
├── poi-cluster-count   → type:symbol   | 聚类计数文本
└── poi-circles         → type:circle   | filter: [!,[has, point_count]]
                                          paint.circle-color:
                                            match(['get','category'],
                                              '景点名胜','#e67e22',
                                              '交通枢纽','#3498db', ...)
```

**数据驱动样式（`match` 表达式）：**
```javascript
'circle-color': [
  'match', ['get','category'],
  '景点名胜', '#e67e22',
  '交通枢纽', '#3498db',
  '餐饮美食', '#e74c3c',
  '购物商圈', '#27ae60',
  '文化教育', '#9b59b6',
  '#999'
]
```

与 Leaflet 端 `makeIcon()` 的颜色完全一致，保证 2D/3D 视图切换时视觉连续性。

### 4.11 样式切换器（3 套配色）

通过运行时 `setPaintProperty` 切换所有图层的 paint 属性，无需重新加载 Style JSON：

| 样式 | 色调 | 底图色 | 水色 |
|------|------|--------|------|
| 街区 (streets) | 暖米 | `#f2efe9` | `#a0c8f0` |
| 暗色 (dark) | 深蓝灰 | `#1e1e2e` | `#1a3a5c` |
| 古风 (vintage) | 羊皮纸 | `#f5ecd7` | `#c8d8c0` |

### 4.12 点聚合可视化

```javascript
maplibreMap.addSource('beijing-pois', {
  type: 'geojson',
  data: fc,
  cluster: true,
  clusterMaxZoom: 14,
  clusterRadius: 50      // 50px 聚类半径
});
```

- 聚类圆大小 `step(point_count, 18, 5,24, 15,32, 30,40)` — 4 级
- 聚类圆颜色 `step` 从黄→红
- 计数文本 `point_count_abbreviated`
- 地图缩放到 14+ 级自动展开为独立圆点

### 4.13 建筑 3D 拉伸（fill-extrusion）

```javascript
{
  id: 'buildings-3d',
  type: 'fill-extrusion',
  source: 'osm-vector',
  'source-layer': 'building',
  minzoom: 14,
  paint: {
    'fill-extrusion-height': [
      'coalesce',
      ['get','height'],           // OSM 显式高度
      ['get','render_height'],    // 预计算渲染高度
      ['*', ['coalesce', ['get','building:levels'], 3], 3]  // 层数×3m 估算
    ],
    'fill-extrusion-color': [
      'interpolate', ['linear'],
      ['coalesce', ['get','height'], ['get','render_height'], ...],
      0, '#d5cfc7', 50, '#b8a090', 150, '#a89888', 300, '#988878'
    ]
  }
}
```

`coalesce` 表达式按优先级回退：`height`（精确值） → `render_height`（预计算） → `building:levels × 3`（估算）。颜色按高度渐变（浅→深），增强立体感。

---

## 五、数据说明

### 5.1 POI 数据

| 文件 | 格式 | 记录数 | 字段 |
|------|------|--------|------|
| `data/beijing-pois.geojson` | GeoJSON FeatureCollection | 20 | name, category, address, description, rating, hours |

### 5.2 行政区划数据

| 文件 | 格式 | 记录数 | 字段 |
|------|------|--------|------|
| `data/beijing-districts.geojson` | GeoJSON FeatureCollection | 16 | name, pop (万人), area (km²), gdp (亿元) |

| 区名 | 人口(万) | 面积(km²) | 人口密度(人/km²) | GDP(亿元) |
|------|:-------:|:-------:|:---------------:|:--------:|
| 东城区 | 70.4 | 41.84 | 16,830 | 3,386 |
| 西城区 | 110.0 | 50.70 | 21,697 | 5,011 |
| 朝阳区 | 345.0 | 470.8 | 7,329 | 8,396 |
| 海淀区 | 313.0 | 430.8 | 7,265 | 10,207 |
| 丰台区 | 202.0 | 305.8 | 6,606 | 2,187 |
| 通州区 | 184.0 | 906.3 | 2,030 | 1,302 |
| 大兴区 | 199.0 | 1036.3 | 1,920 | 1,472 |
| 昌平区 | 228.0 | 1343.5 | 1,697 | 1,402 |
| 顺义区 | 133.0 | 1021.0 | 1,303 | 2,193 |
| 房山区 | 119.0 | 1989.5 | 598 | 873 |
| 石景山区 | 56.8 | 84.38 | 6,733 | 971 |
| 门头沟区 | 35.3 | 1450.7 | 243 | 277 |
| 怀柔区 | 41.0 | 2128.7 | 193 | 463 |
| 密云区 | 52.0 | 2229.5 | 233 | 402 |
| 平谷区 | 45.5 | 1075.0 | 423 | 354 |
| 延庆区 | 35.6 | 1993.8 | 179 | 219 |

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
├── index.html                    # 主页面（HTML + CSS + JS，1380+ 行）
├── data/
│   ├── beijing-pois.geojson      # 北京 20 个 POI 空间数据
│   └── beijing-districts.geojson # 北京 16 区行政区划（含人口/面积/GDP）
├── report-lecture10.md           # 第10讲专题地图实验报告
└── README.md                     # 综合实验报告（本文件）
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
