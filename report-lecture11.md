# 第 11 讲实验报告：MapLibre GL JS — 矢量瓦片、数据驱动样式、聚合与 3D 可视化

---

## 一、实验目的

1. 创建 MapLibre 地图，加载**矢量瓦片底图**（Style JSON）
2. 添加 **GeoJSON Source/Layer**，实现**数据驱动样式**
3. 实现**样式切换器**（多套配色/底图切换）
4. 实现**点聚合可视化**
5. 使用 **fill-extrusion** 实现建筑 3D 拉伸

---

## 二、矢量瓦片底图（Style JSON）

### 2.1 数据源配置

```javascript
const VT_URL = 'https://tiles.openfreemap.org/planet/{z}/{x}/{y}.mvt';

sources: {
  'osm-vector': {
    type: 'vector',
    tiles: [VT_URL],
    maxzoom: 14
  }
}
```

**为何选矢量瓦片而非栅格瓦片？**

| 对比维度 | 栅格瓦片 | 矢量瓦片 |
|---------|:------:|:------:|
| 客户端可定制样式 | ✗ 图片已渲染 | ✅ 每个图层可独立 paint |
| 数据驱动样式 | ✗ | ✅ match/interpolate/step 表达式 |
| 3D 能力 | ✗ | ✅ fill-extrusion |
| 传输体积 | 较大 | 较小（PBF 二进制编码） |
| 旋转无变形 | ✗ 文字跟随旋转 | ✅ 文字始终正立 |

### 2.2 图层架构（12 层）

```
底层
├── background           (background)  底色填充
├── landuse-residential  (fill)        住宅用地
├── landuse-park         (fill)        公园绿地
├── landuse-hospital     (fill)        医院
├── landuse-school       (fill)        学校
├── water                (fill)        水域
├── water-outline        (line)        水域边线
├── road-minor           (line)        支路
├── road-major           (line)        主干道
├── buildings-3d         (fill-extrusion)  ★ 3D 建筑拉伸
├── place-labels         (symbol)      地名标注
└── road-labels          (symbol)      道路名标注
顶层
```

每层通过 `source-layer` 属性指定 OSM 矢量瓦片中的对应数据层（如 `water`、`building`、`transportation`）。

### 2.3 矢量瓦片的 source-layer

OTA 矢量瓦片（OpenMapTiles Schema）标准的分层结构：

| source-layer | 几何类型 | 内容 |
|-------------|:------:|------|
| `water` | Polygon | 河流、湖泊、水库 |
| `landuse` | Polygon | 土地利用（住宅/公园/医院/学校等）|
| `transportation` | Line | 道路（按 class 分 motorway/trunk/primary...）|
| `building` | Polygon | 建筑物轮廓（含 height / building:levels 属性）|
| `place` | Point | 地名标注点 |
| `transportation_name` | Line | 道路名称标注线 |

**class 过滤示例：**
```javascript
{ id: 'road-major', 'source-layer': 'transportation',
  filter: ['==', ['get','class'], 'motorway'] }
```

---

## 三、数据驱动样式（GeoJSON Source + Layer）

### 3.1 三层渲染架构

```
beijing-pois (GeoJSON Source)
│
├── poi-clusters         (type: circle)
│   filter: ['has', 'point_count']
│   用途：聚类圆
│   paint: circle-color 按点数量 step 分级
│          circle-radius 按点数量 step 分级
│
├── poi-cluster-count    (type: symbol)
│   filter: ['has', 'point_count']
│   用途：聚类数字标签
│   layout: text-field = point_count_abbreviated
│
└── poi-circles          (type: circle)
    filter: ['!', ['has', 'point_count']]
    用途：单独 POI 圆点
    paint: circle-color = match(category, ...)  ★ 数据驱动
```

### 3.2 match 表达式——按分类着色

```javascript
'circle-color': [
  'match', ['get', 'category'],     // 读取 feature.properties.category
  '景点名胜', '#e67e22',             // 橙色
  '交通枢纽', '#3498db',             // 蓝色
  '餐饮美食', '#e74c3c',             // 红色
  '购物商圈', '#27ae60',             // 绿色
  '文化教育', '#9b59b6',             // 紫色
  '#999'                            // 默认灰色
]
```

`match` 是 MapLibre 数据驱动表达式的核心函数之一，语义等价于 `switch-case`。浏览器端实时求值，无需预生成样式，修改数据后自动重绘。

### 3.3 三种表达式对比

| 表达式 | 语义 | 适用场景 | 本例何处使用 |
|--------|------|---------|------------|
| `match` | 精确匹配分类 | 离散类别（如 POI 分类） | `circle-color` 按 category 着色 |
| `step` | 阶梯分段 | 有序分级（如聚类大小） | cluster `circle-radius` 按 count 分级 |
| `interpolate` | 线性插值 | 连续值渐变（如建筑高度→色） | `fill-extrusion-color` 按 height 渐变 |

```javascript
// step 示例：聚类圆半径
'circle-radius': ['step', ['get','point_count'],
  18,   // 默认（1-4个）：18px
  5, 24,   // 5-14个：24px
  15, 32,  // 15-29个：32px
  30, 40   // ≥30个：40px
]

// interpolate 示例：建筑颜色按高度渐变
'fill-extrusion-color': ['interpolate',['linear'],['get','height'],
  0,   '#d5cfc7',
  50,  '#b8a090',
  150, '#a89888',
  300, '#988878'
]
```

### 3.4 Popup 交互

```javascript
maplibreMap.on('click', 'poi-circles', (e) => {
  const props = e.features[0].properties;
  new maplibregl.Popup()
    .setLngLat(e.lngLat)
    .setHTML(`<h4>${props.name}</h4>...`)
    .addTo(maplibreMap);
});
```

与 Leaflet `bindPopup` 的区别：MapLibre 不在 layer 上绑定，而是通过地图级事件监听 + 特征查询实现，适合大规模数据。

---

## 四、点聚合（Clustering）

### 4.1 聚类配置

```javascript
maplibreMap.addSource('beijing-pois', {
  type: 'geojson',
  data: fc,
  cluster: true,           // 启用聚类
  clusterMaxZoom: 14,      // 超过此级别不再聚合
  clusterRadius: 50        // 聚类半径（像素）
});
```

### 4.2 聚类圆样式

```javascript
'circle-color': [
  'step', ['get','point_count'],
  '#feb24c',  // <5 个：浅橙
  5,  '#fd8d3c',   // 5-14：中橙
  15, '#e94560',   // 15-29：深品红
  30, '#bd0026'    // ≥30：暗红
],
'circle-radius': [
  'step', ['get','point_count'],
  18, 5, 24, 15, 32, 30, 40
]
```

### 4.3 聚类交互行为

- **缩放展开**：地图缩放到 ≥14 级，聚类自动解聚，显示独立 POI 圆点
- **计数标签**：`point_count_abbreviated` 自动格式化（如 1234 → "1.2k"）
- **点击聚类**：MapLibre 默认行为是放大一级（可通过 `clusterMaxZoom` 控制）

### 4.4 聚类算法原理

MapLibre 采用 **网格聚类**（Grid-based Clustering），基于屏幕像素距离：

```
clusterRadius = 50px → 地图上相距 < 50px 的点合并为一个聚类
```

此算法时间复杂度为 O(n)，适合客户端实时渲染。优点是无需预计算、支持动态数据更新；缺点是缩放级别不同时聚类结果不可复现。

---

## 五、样式切换器

### 5.1 三套配色方案

| 方案 | 设计意图 | 底色 | 水域 | 道路 |
|------|---------|------|------|------|
| **街区** (streets) | 接近标准地图，通用性强 | `#f2efe9` | `#a0c8f0` | 白色 |
| **暗色** (dark) | 暗色主题，减少眩光，适合夜间/投影 | `#1e1e2e` | `#1a3a5c` | 深灰 |
| **古风** (vintage) | 仿羊皮纸质感，风格化展示 | `#f5ecd7` | `#c8d8c0` | 米白 |

### 5.2 切换机制

**方式一：运行时 `setPaintProperty`（本项目采用）**

```javascript
function applyMapLibreStyle(styleName) {
  const palette = STYLE_PALETTES[styleName];
  maplibreMap.setPaintProperty('water', 'fill-color', palette.water);
  maplibreMap.setPaintProperty('road-major', 'line-color', palette.roadMajor);
  // ... 逐层更新 9 个图层
}
```

优点：不重建地图，无缝过渡，用户视角不变。

**方式二：`setStyle()` 替换整个 Style JSON**

优点：可更换完全不同的底图（如 OSM → Mapbox Streets）。缺点：重建导致闪烁，状态丢失。

本项目采用方式一，因为 3 套方案共享同一 source-layer 结构，仅颜色不同。

---

## 六、建筑 3D 拉伸（fill-extrusion）

### 6.1 图层定义

```javascript
{
  id: 'buildings-3d',
  type: 'fill-extrusion',
  source: 'osm-vector',
  'source-layer': 'building',
  minzoom: 14,             // 14 级以下不渲染（数据量太大）
  paint: {
    'fill-extrusion-height': [...],
    'fill-extrusion-color': [...],
    'fill-extrusion-opacity': 0.85
  }
}
```

### 6.2 高度提取——coalesce 三级回退

```javascript
'fill-extrusion-height': [
  'coalesce',
  ['get', 'height'],                           // ① OSM 显式 height 字段（米）
  ['get', 'render_height'],                    // ② 预计算渲染高度
  ['*', ['coalesce', ['get', 'building:levels'], 3], 3]  // ③ 楼层数 × 3m
]
```

`coalesce` 语义：返回第一个非 null 值，从左到右逐级回退。

| 建筑 | height | building:levels | 最终取值 |
|------|:------:|:---------------:|:------:|
| 故宫角楼 | 18m | — | 18m（来自 ①）|
| 普通住宅 | — | 6 | 18m（来自 ③: 6×3）|
| 无数据建筑 | — | — | 9m（来自 ③: coalesce 默认 3×3）|

### 6.3 颜色按高度渐变

```javascript
'fill-extrusion-color': [
  'interpolate', ['linear'],
  ['coalesce', ['get','height'], ...],
  0,   '#d5cfc7',    // 矮建筑 = 浅米灰
  50,  '#b8a090',    // 50m = 中棕
  150, '#a89888',    // 150m = 深棕
  300, '#988878'     // 300m+ = 最深棕
]
```

颜色由浅入深增强"低→高"的立体感知，配合地形、天空、雾效形成完整的 3D 场景。

### 6.4 fill-extrusion 与 2D fill 的对比

| 属性 | `type: fill` | `type: fill-extrusion` |
|------|:----------:|:--------------------:|
| 渲染维度 | 2D | 3D |
| 关键属性 | fill-color, fill-opacity | fill-extrusion-height, fill-extrusion-color |
| 地形贴合 | 贴合地面 | 从地面向上拉伸 |
| 俯仰角可见 | 始终可见（但可能被遮挡） | 仅在 pitch > 0 时有三维效果 |
| 性能消耗 | 低 | 高（GPU 几何计算） |

---

## 七、2D/3D 视图联动

### 7.1 坐标同步

```javascript
// Leaflet → MapLibre
const lCenter = leafletMap.getCenter();
maplibreMap.jumpTo({ center: [lCenter.lng, lCenter.lat], zoom: lCenter.zoom - 0.5 });

// MapLibre → 坐标显示（统一在 mousemove）
maplibreMap.on('mousemove', (e) => {
  document.getElementById('coordLng').textContent = e.lngLat.lng.toFixed(6);
  document.getElementById('coordLat').textContent = e.lngLat.lat.toFixed(6);
});
```

两个引擎共用同一套坐标显示、书签系统（`flyToView` 内部判断当前激活视图）。

### 7.2 MapLibre 控制面板

切换到 MapLibre 视图时显示专属控制面板：
- 3 个样式色块 (streets / dark / vintage)
- 建筑 3D 开关 checkbox
- 聚类图例
- 状态文字

切回 Leaflet 视图时自动隐藏，保持侧边栏整洁。

---

## 八、实验总结

| 学习目标 | 掌握情况 | 关键代码 |
|---------|:-------:|---------|
| 矢量瓦片底图加载 | ✅ | `type:'vector', tiles:[VT_URL]` + 12 层 Style JSON |
| GeoJSON Source/Layer | ✅ | `addSource('beijing-pois', {type:'geojson'})` + 3 层渲染 |
| 数据驱动样式 (match) | ✅ | `match(['get','category'], ...)` 按 POI 分类着色 |
| 数据驱动样式 (step) | ✅ | `step(point_count, ...)` 聚类圆分级 |
| 数据驱动样式 (interpolate) | ✅ | `interpolate(height, ...)` 建筑颜色渐变 |
| `coalesce` 表达式 | ✅ | `coalesce(height, render_height, levels*3)` 三级回退 |
| 样式切换器 | ✅ | `setPaintProperty` 逐层更新，3 套配色方案 |
| 点聚合 (cluster) | ✅ | `cluster:true, clusterRadius:50` + 4 级分层样式 |
| fill-extrusion 3D | ✅ | `fill-extrusion-height/color` + 高度渐变色 |
| Popup 交互 | ✅ | `map.on('click', 'poi-circles', ...)` |
| 视角联动 | ✅ | `jumpTo` + 坐标同步 + 书签兼容 |

---

## 参考资料

- MapLibre GL JS 文档：Style Specification, Expressions. https://maplibre.org/maplibre-gl-js-docs/
- OpenMapTiles Schema. https://openmaptiles.org/schema/
- OpenFreeMap. https://openfreemap.org/
- MapLibre GL JS 数据驱动样式表达式. https://maplibre.org/maplibre-gl-js-docs/style-spec/expressions/
