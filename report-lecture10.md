# 第 10 讲实验报告：专题地图——分级设色图与比例符号图

---

## 一、实验目的

1. 加载行政区划 GeoJSON 数据，使用 `L.geoJSON` + `onEachFeature` 绑定地图交互
2. 制作**分级设色图（Choropleth）**：`getColor` 分级函数 + `style` 回调 + 分级图例
3. 制作**比例符号图（Proportional Symbols）**：按属性值映射符号大小
4. 使用 `pointToLayer` 渲染点要素、`filter` 过滤要素
5. 理解并说明配色方案（顺序/发散色带）的选择依据

---

## 二、实验数据

### 2.1 数据来源与结构

| 项目 | 说明 |
|------|------|
| 文件 | `data/beijing-districts.geojson` |
| 类型 | GeoJSON FeatureCollection (Polygon) |
| 记录数 | 16 条（北京市全部市辖区） |
| 属性字段 | `name`（区名）、`pop`（人口，万人）、`area`（面积，km²）、`gdp`（GDP，亿元） |

### 2.2 各区属性一览

| 区名 | 人口(万) | 面积(km²) | 人口密度(人/km²) | GDP(亿元) |
|------|:-------:|:-------:|:---------------:|:--------:|
| 东城区 | 70.4 | 41.84 | 16,830 | 3,386 |
| 西城区 | 110.0 | 50.70 | 21,697 | 5,011 |
| 朝阳区 | 345.0 | 470.8 | 7,329 | 8,396 |
| 海淀区 | 313.0 | 430.8 | 7,265 | 10,207 |
| 丰台区 | 202.0 | 305.8 | 6,606 | 2,187 |
| 石景山区 | 56.8 | 84.38 | 6,733 | 971 |
| 通州区 | 184.0 | 906.3 | 2,030 | 1,302 |
| 大兴区 | 199.0 | 1036.3 | 1,920 | 1,472 |
| 昌平区 | 228.0 | 1343.5 | 1,697 | 1,402 |
| 顺义区 | 133.0 | 1021.0 | 1,303 | 2,193 |
| 房山区 | 119.0 | 1989.5 | 598 | 873 |
| 门头沟区 | 35.3 | 1450.7 | 243 | 277 |
| 怀柔区 | 41.0 | 2128.7 | 193 | 463 |
| 平谷区 | 45.5 | 1075.0 | 423 | 354 |
| 密云区 | 52.0 | 2229.5 | 233 | 402 |
| 延庆区 | 35.6 | 1993.8 | 179 | 219 |

数据特点：城区（东城/西城）人口密度极高（>15,000），远郊区（延庆/怀柔/密云）密度极低（<250），跨度近 120 倍，适合分级设色可视化。

---

## 三、分级设色图（Choropleth Map）

### 3.1 可视化变量

- **映射变量**：人口密度 = pop × 10,000 / area（人/km²）
- **视觉通道**：填充色 fillColor，按密度值分级

### 3.2 getColor 分级函数

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

### 3.3 色带方案：YlOrRd 7 级

| 级别 | 密度范围 (人/km²) | 色值 | 色样 |
|:----:|-------------------|--------|:--:|
| 1 | 0 – 300 | `#ffffb2` | 🟡 |
| 2 | 300 – 1,000 | `#fed976` | 🟨 |
| 3 | 1,000 – 3,000 | `#feb24c` | 🟧 |
| 4 | 3,000 – 8,000 | `#fd8d3c` | 🟠 |
| 5 | 8,000 – 15,000 | `#f03b20` | 🔴 |
| 6 | 15,000 – 25,000 | `#bd0026` | 🔴 |
| 7 | > 25,000 | `#800026` | 🔴 |

### 3.4 style 回调

```javascript
function choroplethStyle(feature) {
  return {
    fillColor: getColor(getDensity(feature)),
    weight: 1.5,
    color: '#555',
    fillOpacity: 0.78
  };
}
```

`style` 回调在 L.geoJSON 遍历每个要素时被调用，返回的样式对象直接应用于多边形 `L.path` 渲染选项。关键设计：
- `fillOpacity: 0.78` — 半透明，确保底图地名隐约可见
- `weight: 1.5, color: '#555'` — 细灰边，避免边界喧宾夺主

### 3.5 onEachFeature 交互绑定

```javascript
function onEachDistrict(feature, layer) {
  const p = feature.properties;
  const density = getDensity(feature);

  // 缓存形心坐标（供比例符号图复用）
  const centroid = layer.getBounds().getCenter();
  districtCentroids[p.name] = [centroid.lat, centroid.lng];

  // (1) Popup：点击弹窗
  layer.bindPopup(
    `<div class="district-popup">
      <h4>${p.name}</h4>
      <table>
        <tr><td>人口</td><td>${p.pop} 万人</td></tr>
        <tr><td>面积</td><td>${p.area} km²</td></tr>
        <tr><td>人口密度</td><td>${Math.round(density)} 人/km²</td></tr>
        <tr><td>GDP</td><td>${p.gdp} 亿元</td></tr>
      </table>
    </div>`,
    { maxWidth: 240 }
  );

  // (2) Tooltip：鼠标悬停
  layer.bindTooltip(
    `<b>${p.name}</b> — ${Math.round(density)} 人/km²`,
    { sticky: true, direction: 'top', opacity: 0.9 }
  );

  // (3) hover 高亮效果
  layer.on({
    mouseover: highlightFeature,   // 白色边框加粗 + bringToFront
    mouseout:  resetHighlight      // 恢复默认样式
  });
}
```

三种交互逐层递进：
- **Tooltip**（悬停即时）— 最小信息量：区名 + 密度值
- **Popup**（点击确认）— 完整属性表：4 个字段
- **高亮**（hover 反馈）— 白色粗边框提供视觉确认，`bringToFront()` 防止被相邻区遮挡

---

## 四、比例符号图（Proportional Symbol Map）

### 4.1 可视化变量

- **映射变量**：GDP（亿元），反映各区经济总量
- **视觉通道**：圆形符号的面积大小

### 4.2 符号尺寸映射函数

```javascript
function getRadius(gdp) {
  return Math.sqrt(gdp) * 0.85;
}
```

**为何使用 √ 变换？**

| 映射方式 | 公式 | 问题 |
|---------|------|------|
| 线性半径 | r ∝ GDP | GDP 大 4 倍的区，圆面积大 16 倍——读者按面积比较，产生严重错觉 |
| 平方根半径 | r ∝ √GDP | 面积 ∝ GDP，比例关系在视觉上保持正确 |

例如：海淀 GDP = 10,207 亿，延庆 GDP = 219 亿，比值 ≈ 46.6 倍。

- 如果用线性半径：r_海淀 / r_延庆 = 46.6，面积比 = 2175 倍（远大于真实值）
- 用 √ 变换后：r_海淀 / r_延庆 ≈ 6.83，面积比 ≈ 46.6 倍（与真实值一致）

这被称为 **Flannery 感知缩放**，补偿了人类视觉对面积的低估倾向（Weber–Fechner 定律）。

### 4.3 pointToLayer 回调

```javascript
function pointToLayer(feature, latlng) {
  const gdp = feature.properties.gdp;
  return L.circleMarker(latlng, {
    radius: getRadius(gdp),
    fillColor: '#e94560',
    color: '#fff',
    weight: 1.5,
    fillOpacity: 0.65
  });
}
```

`pointToLayer` 替代默认的 `L.marker` 渲染器，返回 `L.circleMarker`。关键参数：
- `fillOpacity: 0.65` — 半透明使重叠符号仍可辨识
- `color: '#fff'` — 白边增强与底图的对比度

### 4.4 filter 回调

```javascript
function propSymbolFilter(feature) {
  return feature.properties.gdp > 0;
}
```

`filter` 在 GeoJSON 遍历前筛选要素。当前全部通过（实际数据均有 GDP），可修改为 `gdp > 500` 仅展示经济规模较大的区。

### 4.5 比例符号 Popup + Tooltip

```javascript
function onEachPropSymbol(feature, layer) {
  const p = feature.properties;
  layer.bindPopup(`...`);          // 区名 + GDP + 人口
  layer.bindTooltip(`${p.name} — ${p.gdp} 亿元`, { sticky: true });
}
```

---

## 五、配色方案选择依据

### 5.1 三种色带类型对比

| 色带类型 | 适用场景 | 示例 | 本例是否适用 |
|---------|---------|------|:----------:|
| **顺序 (Sequential)** | 数值从低到高单向变化 | 人口密度、GDP、降雨量 | ✅ |
| **发散 (Diverging)** | 数值有临界中点，双向偏离 | 温度距平（±℃）、收支差 | ✗ |
| **定性 (Qualitative)** | 无序分类 | 土地类型、语言分布 | ✗ |

### 5.2 选择顺序色带的具体理由

1. **语义方向性明确** — 人口密度是典型的"低→高"单向递增变量。低密度（< 300）对应远郊山区，高密度（> 15,000）对应核心城区，不存在"零值中点"或"正负分化"的语义结构，因此排除发散色带。

2. **感知保序性** — YlOrRd 是**单一色相渐变**（黄→橙→红），色调越深、饱和度越高对应数值越大。多色相渐变（如黄→绿→蓝）会引入色相跳跃，打破读者对"颜色深=数值大"的直觉映射。

3. **ColorBrewer 验证** — YlOrRd 7 级是经过可读性检验的配色方案，支持 colorblind-safe 条件下的 6 级分类。来源：ColorBrewer 2.0 (Brewer, 2003)。

4. **分级断点依据** — 采用非等间距手动断点（近似 Jenks Natural Breaks），针对北京城区-郊区密度极端分化的特征，使分类内方差最小化：

   - 等间距（如 0-4000-8000-…）会使 14/16 个区挤在前 2-3 档，几乎无法区分
   - 手动断点保证每一档有 2-3 个区，高密度段（>8000）3 档独立区分核心城区

### 5.3 比例符号辅助色

比例符号统一使用 `#e94560`（品红色），理由：
- 与分级设色图的黄-橙-红暖色系形成区分，避免视觉通道冲突
- 半透明 fill（0.65）使多个重叠的圆仍然可辨识

---

## 六、完整 L.geoJSON 调用链

```
fetch('beijing-districts.geojson')
  │
  ├─→ [分级设色模式]
  │   L.geoJSON(features, {
  │     style: choroplethStyle,         ← 每个要素调用 getColor(density)
  │     onEachFeature: onEachDistrict   ← bindPopup + bindTooltip + hover
  │   }).addTo(choroplethLayer)
  │
  └─→ [比例符号模式]
      L.geoJSON(features, {
        filter: propSymbolFilter,        ← 先过滤
        pointToLayer: pointToLayer,      ← 再渲染为 circleMarker
        onEachFeature: onEachPropSymbol  ← 绑定 Popup/Tooltip
      }).addTo(propSymbolLayer)
```

双重控制互不冲突：
- **侧边栏 Tab**：选择当前激活的专题图层模式
- **右上角 Layer Control**：勾选/取消覆盖层（与底图切换共用）

---

## 七、实验总结

| 学习目标 | 掌握情况 | 关键代码 |
|---------|:-------:|---------|
| `L.geoJSON` 加载行政区数据 | ✅ | `fetch()` → `L.geoJSON(features, {...})` |
| `style` 回调 + `getColor` 分级 | ✅ | 7 级 YlOrRd 顺序色带，每区 `getColor(getDensity(f))` |
| `onEachFeature` 绑定交互 | ✅ | Popup(4字段) + sticky Tooltip + hover 高亮 + bringToFront |
| `pointToLayer` 自定义点渲染 | ✅ | `L.circleMarker` 替代默认 Marker |
| `filter` 要素过滤 | ✅ | `gdp > 0`（可自定义阈值） |
| 比例符号 √ 缩放 | ✅ | `r = √GDP × 0.85`，补偿面积错觉 |
| 配色方案理论 | ✅ | 顺序/发散/定性色带的适用场景与选择依据 |

---

## 参考资料

- Brewer, C. A. (2003). ColorBrewer 2.0. https://colorbrewer2.org/
- Leaflet 1.9.4 文档：L.geoJSON, style, onEachFeature, pointToLayer, filter. https://leafletjs.com/
- Flannery, J. J. (1971). The relative effectiveness of some common graduated point symbols in the presentation of quantitative data.
