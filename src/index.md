# 感染症データ分析

日本における主要感染症の発生動向を分析します。データは国立健康危機管理研究機構より提供されています。

```js
const diseaseColors = {
  "インフルエンザ": "steelblue",
  "COVID-19": "darkred",
  "梅毒": "purple",
  "百日咳": "orange",
  "RSウイルス": "#ff6b6b",
  "感染性胃腸炎": "#4ecdc4",
  "咽頭結膜熱": "#ffe66d",
  "ARI": "#95e1d3",
  "麻しん": "#f38181",
  "風しん": "#aa96da"
};
```

```js
function formatNumber(value) {
  return value == null ? "N/A" : value.toLocaleString("ja-JP", {minimumFractionDigits: 2, maximumFractionDigits: 2});
}

function formatChange(value) {
  if (value == null || !isFinite(value)) return "N/A";
  const sign = value >= 0 ? "+" : "";
  return `${sign}${value.toFixed(2)}%`;
}

function trend(v) {
  return v >= 0.01 ? html`<span class="green">↗︎</span>`
    : v <= -0.01 ? html`<span class="red">↘︎</span>`
    : "→";
}

function heatmapPlot(data, field, title, minValue = null, maxValue = null) {
  const values = data.map(d => d.値).filter(v => v != null);
  const min = minValue ?? d3.min(values);
  const max = maxValue ?? d3.max(values);
  
  return resize((width) =>
    Plot.plot({
      width,
      height: Math.min(width / 3, 400),
      marginLeft: 40,
      marginBottom: 30,
      x: {label: "週"},
      y: {label: null},
      color: {
        type: "linear",
        domain: [min, max],
        range: ["#f0f7ff", "#0066cc"],
        unknown: "#eee"
      },
      marks: [
        Plot.rect(data, {
          x: "週",
          y: "年",
          fill: "値",
          inset: 0.5,
          tip: true,
          title: d => `${d.年}年第${d.週}週: ${d.値.toFixed(2)}`
        }),
        Plot.text(data.filter(d => d.値 > max * 0.8), {
          x: "週",
          y: "年",
          text: d => Math.round(d.値),
          fontSize: 9,
          fill: "white"
        })
      ]
    })
  );
}
```

---

## 🦠 インフルエンザ

```js
const fluData = Array.from(await sql`
  SELECT * FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 = '総数' AND インフルエンザ_定当 IS NOT NULL
  ORDER BY 開始日
`);

const fluLatest = fluData[fluData.length - 1];
const fluPrevWeek = fluData[fluData.length - 2];
const fluPrevYear = fluData[fluData.length - 53];
const flu52Weeks = fluData.slice(-52);
```

```js
function fluCard(latest, prevWeek, prevYear, weeks52) {
  const current = latest.インフルエンザ_定当;
  const diff1 = current - prevWeek.インフルエンザ_定当;
  const diffY = current - prevYear.インフルエンザ_定当;
  const avg52 = d3.mean(weeks52, d => d.インフルエンザ_定当);
  const range = d3.extent(weeks52, d => d.インフルエンザ_定当);
  const pctChange1 = (diff1 / prevWeek.インフルエンザ_定当) * 100;
  const pctChangeY = (diffY / prevYear.インフルエンザ_定当) * 100;

  return html.fragment`
    <h2 style="color: steelblue;">定点当たり報告数</h2>
    <h1>${formatNumber(current)}</h1>
    <span class="small muted">最新週（${latest.年}年第${latest.週}週）</span>
    <table>
      <tr>
        <td>1週間変化</td>
        <td align="right">${formatChange(pctChange1)}</td>
        <td>${trend(pctChange1)}</td>
      </tr>
      <tr>
        <td>前年同週比</td>
        <td align="right">${formatChange(pctChangeY)}</td>
        <td>${trend(pctChangeY)}</td>
      </tr>
      <tr>
        <td>52週平均</td>
        <td align="right">${formatNumber(avg52)}</td>
      </tr>
    </table>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 60,
        axis: null,
        x: {inset: 20},
        marginLeft: 5,
        marginRight: 5,
        marks: [
          Plot.areaY(weeks52, {
            x: (d, i) => i,
            y: d => d.インフルエンザ_定当,
            fill: "steelblue",
            fillOpacity: 0.2
          }),
          Plot.line(weeks52, {
            x: (d, i) => i,
            y: d => d.インフルエンザ_定当,
            stroke: "steelblue",
            strokeWidth: 2
          }),
          Plot.text([formatNumber(range[0])], {frameAnchor: "left", dx: 5, dy: -5}),
          Plot.text([formatNumber(range[1])], {frameAnchor: "right", dx: -5, dy: -5})
        ]
      })
    )}
    <span class="small muted">52週レンジ</span>
  `;
}
```

<style type="text/css">
/* Prevent grid children from forcing horizontal overflow */
.grid > .card {
  min-width: 0;
}

.card {
  min-width: 0;
}

@container (min-width: 640px) {
  .disease-grid {
    grid-template-columns: minmax(0, 1fr) minmax(0, 2fr);
  }
}
</style>

<div class="grid disease-grid">
  <div class="card">${fluCard(fluLatest, fluPrevWeek, fluPrevYear, flu52Weeks)}</div>
  <div class="card">
    <h2>定点当たり報告数の推移（2012年〜）</h2>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 420,
        marginTop: 30,
        marginRight: 20,
        marginBottom: 40,
        marginLeft: 150,
        x: {
          type: "utc",
          label: null
        },
        y: {
          label: "定点当たり",
          grid: true
        },
        marks: [
          Plot.line(fluData, {
            x: d => new Date(d.開始日),
            y: "インフルエンザ_定当",
            stroke: "steelblue",
            strokeWidth: 2,
            tip: true,
            title: d => `${d.年}年第${d.週}週 (${d.開始日}): ${d.インフルエンザ_定当}`
          }),
          Plot.ruleY([0]),
          Plot.ruleY([30], {stroke: "red", strokeDasharray: "4,4", strokeOpacity: 0.5}),
          Plot.text(["警報レベル"], {x: new Date("2020-01-01"), y: 30, dy: -8, fill: "red", fontSize: 10})
        ]
      })
    )}
  </div>
</div>

### 主要都道府県比較（2020年以降）

```js
const fluPrefectures = Array.from(await sql`
  SELECT * FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 IN ('東京都', '大阪府', '北海道', '福岡県', '愛知県', '沖縄県')
    AND インフルエンザ_定当 IS NOT NULL
    AND 年 >= 2020
  ORDER BY 開始日
`);
```

<div class="grid grid-cols-1">
  <div class="card">
    ${resize((width) =>
      Plot.plot({
        width,
        height: 400,
        x: {type: "utc", label: null},
        y: {label: "定点当たり報告数", grid: true},
        marks: [
          Plot.line(fluPrefectures, {
            x: d => new Date(d.開始日),
            y: "インフルエンザ_定当",
            stroke: "都道府県",
            strokeWidth: 2,
            tip: true,
            title: d => `${d.都道府県} ${d.年}年第${d.週}週 (${d.開始日}): ${d.インフルエンザ_定当}`
          })
        ]
      })
    )}
  </div>
</div>

### 週別熱力図（2015年〜）

```js
const fluHeatmap = Array.from(await sql`
  SELECT 
    年,
    週,
    インフルエンザ_定当 as 値
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 = '総数' AND インフルエンザ_定当 IS NOT NULL AND 年 >= 2015
  ORDER BY 年, 週
`);
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>インフルエンザ - 週別熱力図</h2>
    ${heatmapPlot(fluHeatmap, "インフルエンザ_定当", "インフルエンザ")}
  </div>
</div>

---

## 🦠 COVID-19

```js
const covidData = Array.from(await sql`
  SELECT * FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 = '総数' AND "COVID-19_定当" IS NOT NULL
  ORDER BY 開始日
`);

const covidLatest = covidData[covidData.length - 1];
const covid52Weeks = covidData.slice(-52);
```

```js
function covidCard(latest, weeks52) {
  const current = latest["COVID-19_定当"];
  const avg52 = d3.mean(weeks52, d => d["COVID-19_定当"]);
  const range = d3.extent(weeks52, d => d["COVID-19_定当"]);

  return html.fragment`
    <h2 style="color: darkred;">定点当たり報告数</h2>
    <h1>${formatNumber(current)}</h1>
    <span class="small muted">最新週（${latest.年}年第${latest.週}週）</span>
    <table>
      <tr>
        <td>52週平均</td>
        <td align="right">${formatNumber(avg52)}</td>
      </tr>
    </table>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 60,
        axis: null,
        x: {inset: 20},
        marginLeft: 5,
        marginRight: 5,
        marks: [
          Plot.areaY(weeks52, {
            x: (d, i) => i,
            y: d => d["COVID-19_定当"],
            fill: "darkred",
            fillOpacity: 0.2
          }),
          Plot.line(weeks52, {
            x: (d, i) => i,
            y: d => d["COVID-19_定当"],
            stroke: "darkred",
            strokeWidth: 2
          }),
          Plot.text([formatNumber(range[0])], {frameAnchor: "left", dx: 5, dy: -5}),
          Plot.text([formatNumber(range[1])], {frameAnchor: "right", dx: -5, dy: -5})
        ]
      })
    )}
    <span class="small muted">52週レンジ</span>
  `;
}
```

<div class="grid disease-grid">
  <div class="card">${covidCard(covidLatest, covid52Weeks)}</div>
  <div class="card">
    <h2>定点当たり報告数の推移</h2>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 420,
        marginTop: 30,
        marginRight: 20,
        marginBottom: 40,
        marginLeft: 150,
        x: {type: "utc", label: null},
        y: {label: "定点当たり", grid: true},
        marks: [
          Plot.line(covidData, {
            x: d => new Date(d.開始日),
            y: "COVID-19_定当",
            stroke: "darkred",
            strokeWidth: 2,
            tip: true,
            title: d => `${d.年}年第${d.週}週 (${d.開始日}): ${d["COVID-19_定当"]}`
          }),
          Plot.ruleY([0])
        ]
      })
    )}
  </div>
</div>

### 週別熱力図（2020年〜）

```js
const covidHeatmap = Array.from(await sql`
  SELECT 
    年,
    週,
    "COVID-19_定当" as 値
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 = '総数' AND "COVID-19_定当" IS NOT NULL AND 年 >= 2020
  ORDER BY 年, 週
`);
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>COVID-19 - 週別熱力図</h2>
    ${heatmapPlot(covidHeatmap, "COVID-19_定当", "COVID-19")}
  </div>
</div>

---

## 🦠 梅毒

```js
const syphilisYearly = Array.from(await sql`
  SELECT 
    年,
    SUM(梅毒_報告) as 年間総報告数
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/zensu/merged_zensu.parquet')
  WHERE 都道府県 = '総数' AND 梅毒_報告 IS NOT NULL
  GROUP BY 年
  ORDER BY 年
`);

const syphilisLatest = syphilisYearly[syphilisYearly.length - 1];
```

```js
function syphilisCard(latest, yearly) {
  const current = latest.年間総報告数;
  const avg = d3.mean(yearly, d => d.年間総報告数);
  const range = d3.extent(yearly, d => d.年間総報告数);

  return html.fragment`
    <h2 style="color: purple;">年間報告数</h2>
    <h1>${current.toLocaleString()}</h1>
    <span class="small muted">${latest.年}年累積</span>
    <table>
      <tr>
        <td>平均（全期間）</td>
        <td align="right">${Math.round(avg).toLocaleString()}</td>
      </tr>
    </table>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 60,
        axis: null,
        x: {inset: 20},
        marginLeft: 5,
        marginRight: 5,
        marks: [
          Plot.areaY(yearly, {
            x: "年",
            y: "年間総報告数",
            fill: "purple",
            fillOpacity: 0.2
          }),
          Plot.line(yearly, {
            x: "年",
            y: "年間総報告数",
            stroke: "purple",
            strokeWidth: 2
          }),
          Plot.text([range[0].toLocaleString()], {frameAnchor: "left", dx: 5, dy: -5}),
          Plot.text([range[1].toLocaleString()], {frameAnchor: "right", dx: -5, dy: -5})
        ]
      })
    )}
    <span class="small muted">全期間レンジ</span>
  `;
}
```

<div class="grid disease-grid">
  <div class="card">${syphilisCard(syphilisLatest, syphilisYearly)}</div>
  <div class="card">
    <h2>年間報告数の推移</h2>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 360,
        marginTop: 30,
        marginRight: 20,
        marginBottom: 40,
        marginLeft: 150,
        x: {label: null},
        y: {
          label: "年間報告数",
          grid: true,
          tickFormat: d => d >= 1000 ? `${(d/1000).toFixed(0)}k` : d
        },
        marks: [
          Plot.barY(syphilisYearly, {
            x: "年",
            y: "年間総報告数",
            fill: "purple",
            tip: true,
            title: d => `${d.年}: ${d.年間総報告数}`
          }),
          Plot.ruleY([0])
        ]
      })
    )}
  </div>
</div>

### 週別熱力図（2015年〜）

```js
const syphilisHeatmap = Array.from(await sql`
  SELECT 
    年,
    週,
    梅毒_報告 as 値
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/zensu/merged_zensu.parquet')
  WHERE 都道府県 = '総数' AND 梅毒_報告 IS NOT NULL AND 年 >= 2015
  ORDER BY 年, 週
`);
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>梅毒 - 週別熱力図（週報告数）</h2>
    ${heatmapPlot(syphilisHeatmap, "梅毒_報告", "梅毒")}
  </div>
</div>

```js
const pertussisYearly = Array.from(await sql`
  SELECT 
    年,
    SUM(百日咳_報告) as 年間総報告数
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/zensu/merged_zensu.parquet')
  WHERE 都道府県 = '総数' AND 百日咳_報告 IS NOT NULL
  GROUP BY 年
  ORDER BY 年
`);

const pertussisLatest = pertussisYearly[pertussisYearly.length - 1];
```

```js
function pertussisCard(latest, yearly) {
  const current = latest.年間総報告数;
  const avg = d3.mean(yearly, d => d.年間総報告数);
  const range = d3.extent(yearly, d => d.年間総報告数);

  return html.fragment`
    <h2 style="color: orange;">年間報告数</h2>
    <h1>${current.toLocaleString()}</h1>
    <span class="small muted">${latest.年}年累積</span>
    <table>
      <tr>
        <td>平均（全期間）</td>
        <td align="right">${Math.round(avg).toLocaleString()}</td>
      </tr>
    </table>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 60,
        axis: null,
        x: {inset: 20},
        marginLeft: 5,
        marginRight: 5,
        marks: [
          Plot.areaY(yearly, {
            x: "年",
            y: "年間総報告数",
            fill: "orange",
            fillOpacity: 0.2
          }),
          Plot.line(yearly, {
            x: "年",
            y: "年間総報告数",
            stroke: "orange",
            strokeWidth: 2
          }),
          Plot.text([range[0].toLocaleString()], {frameAnchor: "left", dx: 5, dy: -5}),
          Plot.text([range[1].toLocaleString()], {frameAnchor: "right", dx: -5, dy: -5})
        ]
      })
    )}
    <span class="small muted">全期間レンジ</span>
  `;
}
```

<div class="grid disease-grid">
  <div class="card">${pertussisCard(pertussisLatest, pertussisYearly)}</div>
  <div class="card">
    <h2>年間報告数の推移</h2>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 360,
        marginTop: 30,
        marginRight: 20,
        marginBottom: 40,
        marginLeft: 150,
        x: {label: null},
        y: {
          label: "年間報告数",
          grid: true,
          tickFormat: d => d >= 1000 ? `${(d/1000).toFixed(0)}k` : d
        },
        marks: [
          Plot.barY(pertussisYearly, {
            x: "年",
            y: "年間総報告数",
            fill: "orange",
            tip: true,
            title: d => `${d.年}: ${d.年間総報告数}`
          }),
          Plot.ruleY([0])
        ]
      })
    )}
  </div>
</div>

### 週別熱力図（2015年〜）

```js
const pertussisHeatmap = Array.from(await sql`
  SELECT 
    年,
    週,
    百日咳_報告 as 値
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/zensu/merged_zensu.parquet')
  WHERE 都道府県 = '総数' AND 百日咳_報告 IS NOT NULL AND 年 >= 2015
  ORDER BY 年, 週
`);
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>百日咳 - 週別熱力図（週報告数）</h2>
    ${heatmapPlot(pertussisHeatmap, "百日咳_報告", "百日咳")}
  </div>
</div>

---

## 🦠 RSウイルス感染症

```js
const rsData = Array.from(await sql`
  SELECT * FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 = '総数' AND ＲＳウイルス感染症_定当 IS NOT NULL AND 年 >= 2020
  ORDER BY 開始日
`);

const rsLatest = rsData[rsData.length - 1];
const rsPrevWeek = rsData[rsData.length - 2];
const rs52Weeks = rsData.slice(-52);
```

```js
function rsCard(latest, prevWeek, weeks52) {
  const current = latest.ＲＳウイルス感染症_定当;
  const diff1 = current - prevWeek.ＲＳウイルス感染症_定当;
  const avg52 = d3.mean(weeks52, d => d.ＲＳウイルス感染症_定当);
  const range = d3.extent(weeks52, d => d.ＲＳウイルス感染症_定当);
  const pctChange1 = (diff1 / prevWeek.ＲＳウイルス感染症_定当) * 100;

  return html.fragment`
    <h2 style="color: #ff6b6b;">定点当たり報告数</h2>
    <h1>${formatNumber(current)}</h1>
    <span class="small muted">最新週（${latest.年}年第${latest.週}週）</span>
    <table>
      <tr>
        <td>1週間変化</td>
        <td align="right">${formatChange(pctChange1)}</td>
        <td>${trend(pctChange1)}</td>
      </tr>
      <tr>
        <td>52週平均</td>
        <td align="right">${formatNumber(avg52)}</td>
      </tr>
    </table>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 60,
        axis: null,
        x: {inset: 20},
        marginLeft: 5,
        marginRight: 5,
        marks: [
          Plot.areaY(weeks52, {
            x: (d, i) => i,
            y: d => d.ＲＳウイルス感染症_定当,
            fill: "#ff6b6b",
            fillOpacity: 0.2
          }),
          Plot.line(weeks52, {
            x: (d, i) => i,
            y: d => d.ＲＳウイルス感染症_定当,
            stroke: "#ff6b6b",
            strokeWidth: 2
          }),
          Plot.text([formatNumber(range[0])], {frameAnchor: "left", dx: 5, dy: -5}),
          Plot.text([formatNumber(range[1])], {frameAnchor: "right", dx: -5, dy: -5})
        ]
      })
    )}
    <span class="small muted">52週レンジ</span>
  `;
}
```

<div class="grid disease-grid">
  <div class="card">${rsCard(rsLatest, rsPrevWeek, rs52Weeks)}</div>
  <div class="card">
    <h2>定点当たり報告数の推移（2020年〜）</h2>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 420,
        marginTop: 30,
        marginRight: 20,
        marginBottom: 40,
        marginLeft: 150,
        x: {type: "utc", label: null},
        y: {label: "定点当たり", grid: true},
        marks: [
          Plot.line(rsData, {
            x: d => new Date(d.開始日),
            y: "ＲＳウイルス感染症_定当",
            stroke: "#ff6b6b",
            strokeWidth: 2,
            tip: true,
            title: d => `${d.年}年第${d.週}週 (${d.開始日}): ${d.ＲＳウイルス感染症_定当}`
          }),
          Plot.ruleY([0])
        ]
      })
    )}
  </div>
</div>

### 週別熱力図（2020年〜）

```js
const rsHeatmap = Array.from(await sql`
  SELECT 
    年,
    週,
    ＲＳウイルス感染症_定当 as 値
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 = '総数' AND ＲＳウイルス感染症_定当 IS NOT NULL AND 年 >= 2020
  ORDER BY 年, 週
`);
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>RSウイルス感染症 - 週別熱力図</h2>
    ${heatmapPlot(rsHeatmap, "ＲＳウイルス感染症_定当", "RSウイルス")}
  </div>
</div>

---

## 🦠 感染性胃腸炎

```js
const gastroData = Array.from(await sql`
  SELECT * FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 = '総数' AND 感染性胃腸炎_定当 IS NOT NULL AND 年 >= 2020
  ORDER BY 開始日
`);

const gastroLatest = gastroData[gastroData.length - 1];
const gastroPrevWeek = gastroData[gastroData.length - 2];
const gastro52Weeks = gastroData.slice(-52);
```

```js
function gastroCard(latest, prevWeek, weeks52) {
  const current = latest.感染性胃腸炎_定当;
  const diff1 = current - prevWeek.感染性胃腸炎_定当;
  const avg52 = d3.mean(weeks52, d => d.感染性胃腸炎_定当);
  const range = d3.extent(weeks52, d => d.感染性胃腸炎_定当);
  const pctChange1 = (diff1 / prevWeek.感染性胃腸炎_定当) * 100;

  return html.fragment`
    <h2 style="color: #4ecdc4;">定点当たり報告数</h2>
    <h1>${formatNumber(current)}</h1>
    <span class="small muted">最新週（${latest.年}年第${latest.週}週）</span>
    <table>
      <tr>
        <td>1週間変化</td>
        <td align="right">${formatChange(pctChange1)}</td>
        <td>${trend(pctChange1)}</td>
      </tr>
      <tr>
        <td>52週平均</td>
        <td align="right">${formatNumber(avg52)}</td>
      </tr>
    </table>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 60,
        axis: null,
        x: {inset: 20},
        marginLeft: 5,
        marginRight: 5,
        marks: [
          Plot.areaY(weeks52, {
            x: (d, i) => i,
            y: d => d.感染性胃腸炎_定当,
            fill: "#4ecdc4",
            fillOpacity: 0.2
          }),
          Plot.line(weeks52, {
            x: (d, i) => i,
            y: d => d.感染性胃腸炎_定当,
            stroke: "#4ecdc4",
            strokeWidth: 2
          }),
          Plot.text([formatNumber(range[0])], {frameAnchor: "left", dx: 5, dy: -5}),
          Plot.text([formatNumber(range[1])], {frameAnchor: "right", dx: -5, dy: -5})
        ]
      })
    )}
    <span class="small muted">52週レンジ</span>
  `;
}
```

<div class="grid disease-grid">
  <div class="card">${gastroCard(gastroLatest, gastroPrevWeek, gastro52Weeks)}</div>
  <div class="card">
    <h2>定点当たり報告数の推移（2020年〜）</h2>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 420,
        marginTop: 30,
        marginRight: 20,
        marginBottom: 40,
        marginLeft: 150,
        x: {type: "utc", label: null},
        y: {label: "定点当たり", grid: true},
        marks: [
          Plot.line(gastroData, {
            x: d => new Date(d.開始日),
            y: "感染性胃腸炎_定当",
            stroke: "#4ecdc4",
            strokeWidth: 2,
            tip: true,
            title: d => `${d.年}年第${d.週}週 (${d.開始日}): ${d.感染性胃腸炎_定当}`
          }),
          Plot.ruleY([0])
        ]
      })
    )}
  </div>
</div>

### 週別熱力図（2020年〜）

```js
const gastroHeatmap = Array.from(await sql`
  SELECT 
    年,
    週,
    感染性胃腸炎_定当 as 値
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 = '総数' AND 感染性胃腸炎_定当 IS NOT NULL AND 年 >= 2020
  ORDER BY 年, 週
`);
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>感染性胃腸炎 - 週別熱力図</h2>
    ${heatmapPlot(gastroHeatmap, "感染性胃腸炎_定当", "感染性胃腸炎")}
  </div>
</div>

---

## 🦠 急性呼吸器感染症（ARI）

```js
const ariDataFiltered = Array.from(await sql`
  SELECT * FROM read_parquet('https://kansenshou.ringsaturn.me/data/ari/merged_ari.parquet')
  WHERE 都道府県 = '総数' AND 急性呼吸器感染症_定当 IS NOT NULL
  ORDER BY 開始日
`);

const ariLatest = ariDataFiltered[ariDataFiltered.length - 1];
const ariPrevWeek = ariDataFiltered[ariDataFiltered.length - 2];
```

```js
function ariCard(latest, prevWeek, allData) {
  const current = latest.急性呼吸器感染症_定当;
  const diff1 = current - prevWeek.急性呼吸器感染症_定当;
  const avg = d3.mean(allData, d => d.急性呼吸器感染症_定当);
  const range = d3.extent(allData, d => d.急性呼吸器感染症_定当);
  const pctChange1 = (diff1 / prevWeek.急性呼吸器感染症_定当) * 100;

  return html.fragment`
    <h2 style="color: #95e1d3;">定点当たり報告数</h2>
    <h1>${formatNumber(current)}</h1>
    <span class="small muted">最新週（${latest.年}年第${latest.週}週）</span>
    <table>
      <tr>
        <td>1週間変化</td>
        <td align="right">${formatChange(pctChange1)}</td>
        <td>${trend(pctChange1)}</td>
      </tr>
      <tr>
        <td>全期間平均</td>
        <td align="right">${formatNumber(avg)}</td>
      </tr>
      <tr>
        <td>報告数</td>
        <td align="right">${latest.急性呼吸器感染症_報告.toLocaleString()}</td>
      </tr>
    </table>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 60,
        axis: null,
        x: {inset: 20},
        marginLeft: 5,
        marginRight: 5,
        marks: [
          Plot.areaY(allData, {
            x: (d, i) => i,
            y: d => d.急性呼吸器感染症_定当,
            fill: "#95e1d3",
            fillOpacity: 0.2
          }),
          Plot.line(allData, {
            x: (d, i) => i,
            y: d => d.急性呼吸器感染症_定当,
            stroke: "#95e1d3",
            strokeWidth: 2
          }),
          Plot.text([formatNumber(range[0])], {frameAnchor: "left", dx: 5, dy: -5}),
          Plot.text([formatNumber(range[1])], {frameAnchor: "right", dx: -5, dy: -5})
        ]
      })
    )}
    <span class="small muted">全期間レンジ</span>
  `;
}
```

<div class="grid disease-grid">
  <div class="card">${ariCard(ariLatest, ariPrevWeek, ariDataFiltered)}</div>
  <div class="card">
    <h2>定点当たり報告数の推移（2025年〜）</h2>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 420,
        marginTop: 30,
        marginRight: 20,
        marginBottom: 40,
        marginLeft: 150,
        x: {type: "utc", label: null},
        y: {label: "定点当たり", grid: true},
        marks: [
          Plot.line(ariDataFiltered, {
            x: d => new Date(d.開始日),
            y: "急性呼吸器感染症_定当",
            stroke: "#95e1d3",
            strokeWidth: 2,
            tip: true,
            title: d => `${d.年}年第${d.週}週 (${d.開始日}): ${d.急性呼吸器感染症_定当}`
          }),
          Plot.ruleY([0])
        ]
      })
    )}
  </div>
</div>

### 週別熱力図（2025年）

```js
const ariHeatmap = Array.from(await sql`
  SELECT 
    年,
    週,
    急性呼吸器感染症_定当 as 値
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/ari/merged_ari.parquet')
  WHERE 都道府県 = '総数' AND 急性呼吸器感染症_定当 IS NOT NULL
  ORDER BY 年, 週
`);
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>急性呼吸器感染症（ARI）- 週別熱力図</h2>
    ${heatmapPlot(ariHeatmap, "急性呼吸器感染症_定当", "ARI")}
  </div>
</div>

---

## 🦠 麻しん・風しん

```js
const measlesYearly = Array.from(await sql`
  SELECT 
    年,
    SUM(麻しん_報告) as 麻しん,
    SUM(風しん_報告) as 風しん
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/zensu/merged_zensu.parquet')
  WHERE 都道府県 = '総数' AND 麻しん_報告 IS NOT NULL AND 年 >= 2012
  GROUP BY 年
  ORDER BY 年
`);

const measlesLatest = measlesYearly[measlesYearly.length - 1];
```

<div class="grid grid-cols-2">
  <div class="card">
    <h2 style="color: #f38181;">麻しん 年間報告数</h2>
    <h1>${measlesLatest.麻しん.toLocaleString()}</h1>
    <span class="small muted">${measlesLatest.年}年累積</span>
  </div>
  <div class="card">
    <h2 style="color: #aa96da;">風しん 年間報告数</h2>
    <h1>${measlesLatest.風しん.toLocaleString()}</h1>
    <span class="small muted">${measlesLatest.年}年累積</span>
  </div>
</div>

<div class="grid grid-cols-1">
  <div class="card">
    <h2>年間報告数の推移（2012年〜）</h2>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 400,
        marginTop: 30,
        marginRight: 20,
        marginBottom: 40,
        marginLeft: 150,
        x: {label: null},
        y: {label: "年間報告数", grid: true},
        color: {
          domain: ["麻しん", "風しん"],
          range: ["#f38181", "#aa96da"]
        },
        marks: [
          Plot.line(measlesYearly, {
            x: "年",
            y: "麻しん",
            stroke: "#f38181",
            strokeWidth: 2,
            tip: true,
            title: d => `麻しん ${d.年}: ${d.麻しん}`
          }),
          Plot.line(measlesYearly, {
            x: "年",
            y: "風しん",
            stroke: "#aa96da",
            strokeWidth: 2,
            tip: true,
            title: d => `風しん ${d.年}: ${d.風しん}`
          }),
          Plot.dot(measlesYearly, {
            x: "年",
            y: "麻しん",
            fill: "#f38181",
            r: 3
          }),
          Plot.dot(measlesYearly, {
            x: "年",
            y: "風しん",
            fill: "#aa96da",
            r: 3
          }),
          Plot.ruleY([0])
        ]
      })
    )}
  </div>
</div>

---

## 📊 多疾病比較：インフルエンザ vs COVID-19（2020年以降）

```js
const comparison = Array.from(await sql`
  SELECT 
    開始日::TIMESTAMP as date,
    インフルエンザ_定当 as value,
    'インフルエンザ' as type
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 = '総数' AND インフルエンザ_定当 IS NOT NULL AND 年 >= 2020
  
  UNION ALL
  
  SELECT 
    開始日::TIMESTAMP as date,
    "COVID-19_定当" as value,
    'COVID-19' as type
  FROM read_parquet('https://kansenshou.ringsaturn.me/data/teiten/merged_teiten.parquet')
  WHERE 都道府県 = '総数' AND "COVID-19_定当" IS NOT NULL AND 年 >= 2020
  
  ORDER BY date
`);
```

<div class="grid grid-cols-1">
  <div class="card">
    <h2>定点当たり報告数の比較</h2>
    ${resize((width) =>
      Plot.plot({
        width,
        height: 500,
        x: {type: "utc", label: null},
        y: {label: "定点当たり報告数", grid: true},
        color: {
          domain: ["インフルエンザ", "COVID-19"],
          range: ["steelblue", "darkred"]
        },
        marks: [
          Plot.line(comparison, {
            x: "date",
            y: "value",
            stroke: "type",
            strokeWidth: 2,
            tip: true,
            title: d => `${d.type} ${d.date.toLocaleDateString("ja-JP")}: ${d.value}`
          })
        ]
      })
    )}
  </div>
</div>

---

## 📝 データソース

**出典**: 国立健康危機管理研究機構 感染症情報提供サイト
https://id-info.jihs.go.jp/surveillance/idwr/

データの利用については原サイトの利用規約をご確認ください。
