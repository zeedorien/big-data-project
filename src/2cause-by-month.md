---
toc: false
---

<style>
  .chart-container {
    height: 560px;        /* matches chart height */
    position: relative;
  }
  .chart-svg {
    transition: opacity 0.2s ease;
  }
</style>

# Flight Delays by Causes and Month

---

## Analysis of Months (agg. June 2003 - November 2025)

The next angle examined was to understand how delay causes vary across the calendar year. A multi-line chart was created to track aggregated monthly delay minutes for the five categories included in the dataset: Carrier Delays, Weather, Traffic (NAS) Delays, Late Aircraft, and Security. The visualization reveals a seasonal pattern, where total delays peak in June and July, decline through the fall, and remain relatively low during winter months, with a small peak in the Winter months. This matches explanations for flight volume (and therefore delays) to be highest during the summer months and holidays. 

Late Aircraft emerges as the dominant contributor across all months, followed by Carrier Delays, while Weather accounts for a smaller share despite the expectation that storms would drive higher delays. Security delays is the smallest cause as expected, as security abnormalities are rare in commercial aviation. 

The tooltip that appears when hovering over the line chart shows the composition of delays for any given month. In general, it appears the proportion of delay causes remains steady across all months, with only total delay volume fluctuating. The breakdown also suggests that operational factors like aircraft turnaround times and airline scheduling decisions have the greatest effect on airline delays.

```js

// Monthly delay lines - aggregated across all years
import * as d3 from "npm:d3";
import {resize, html} from "npm:@observablehq/stdlib";

const flightData = await FileAttachment("data/flights.csv").csv({typed: true});

// Aggregate all rows by month (ignoring year)
const monthlyMap = new Map(); // key: month number

for (const row of flightData) {
  const month = +row.month;
  if (!monthlyMap.has(month)) {
    monthlyMap.set(month, {
      month,
      carrierDelay: 0,
      weatherDelay: 0,
      nasDelay: 0,
      securityDelay: 0,
      lateAircraftDelay: 0,
      totalFlights: 0
    });
  }
  const agg = monthlyMap.get(month);
  agg.carrierDelay += +row.carrier_delay || 0;
  agg.weatherDelay += +row.weather_delay || 0;
  agg.nasDelay += +row.nas_delay || 0;
  agg.securityDelay += +row.security_delay || 0;
  agg.lateAircraftDelay += +row.late_aircraft_delay || 0;
  agg.totalFlights += +row.arr_flights || 0;
}

// Convert to array and sort by month
let monthlyData = Array.from(monthlyMap.values());
monthlyData.sort((a,b) => a.month - b.month);

// Compute total delay per month
for (const d of monthlyData) {
  d.totalDelay = d.carrierDelay + d.weatherDelay + d.nasDelay + d.securityDelay + d.lateAircraftDelay;
}

// Check if we have data
if (monthlyData.length === 0) {
  display(html`<div style="color:red;">No monthly data found.</div>`);
} else {
  // Chart function (similar structure but using yearly aggregated data)
  function monthlyAggChart(width) {
    const height = 560;
    const margin = { top: 80, right: 230, bottom: 80, left: 125 };
    const months = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"];

    // Ensure all months 1..12 exist
    const data = [];
    for (let m = 1; m <= 12; m++) {
      const row = monthlyData.find(d => d.month === m);
      data.push(row || { month: m, carrierDelay:0, weatherDelay:0, nasDelay:0, securityDelay:0, lateAircraftDelay:0, totalDelay:0, totalFlights:0 });
    }

    const x = d3.scalePoint()
      .domain(data.map(d => d.month))
      .range([margin.left, width - margin.right])
      .padding(0.25);

    const maxY = d3.max(data, d => Math.max(d.totalDelay, d.carrierDelay, d.weatherDelay, d.nasDelay, d.securityDelay, d.lateAircraftDelay));
    const y = d3.scaleLinear()
      .domain([0, maxY * 1.12])
      .range([height - margin.bottom, margin.top]);

    const line = d3.line()
      .curve(d3.curveLinear)   // straight lines between points
      .x(d => x(d.month))
      .y(d => y(d.value));

    const series = [
      { key: "total", name: "Total", color: "#2f6b1f", strokeWidth: 5,
        values: data.map(d => ({ month: d.month, value: d.totalDelay })) },
      { key: "carrierDelay", name: "Carrier Delays", color: "#7b3294", strokeWidth: 3,
        values: data.map(d => ({ month: d.month, value: d.carrierDelay })) },
      { key: "weatherDelay", name: "Weather", color: "#e31a1c", strokeWidth: 3,
        values: data.map(d => ({ month: d.month, value: d.weatherDelay })) },
      { key: "nasDelay", name: "Traffic (NAS) Delays", color: "#f39c12", strokeWidth: 3,
        values: data.map(d => ({ month: d.month, value: d.nasDelay })) },
      { key: "lateAircraftDelay", name: "Late Aircraft", color: "#1f78b4", strokeWidth: 3,
        values: data.map(d => ({ month: d.month, value: d.lateAircraftDelay })) },
      { key: "securityDelay", name: "Security", color: "#c51b7d", strokeWidth: 3,
        values: data.map(d => ({ month: d.month, value: d.securityDelay })) }
    ];

    const svg = d3.create("svg")
      .attr("width", width)
      .attr("height", height)
      .style("overflow", "visible");

    // Title (indicate it's aggregated across all years)
    svg.append("text")
      .attr("x", width / 2).attr("y", 40)
      .attr("text-anchor", "middle")
      .attr("font-size", 24).attr("font-weight", 700)
      .attr("fill", "white")
      .text("Monthly Airline Delay Minutes by Cause (Aggregated)");

    // Axes
    svg.append("g")
      .attr("transform", `translate(0,${height - margin.bottom})`)
      .call(d3.axisBottom(x).tickFormat(m => months[m - 1]))
      .selectAll("text").attr("font-size", 14);

    svg.append("g")
        .attr("transform", `translate(${margin.left},0)`)
        .call(d3.axisLeft(y).tickFormat(d => (d / 1e6).toFixed(0) + "m"))
        .selectAll("text")
        .attr("font-size", 14)
        .attr("fill", "white");
      

    // Axis labels
    svg.append("text")
      .attr("x", width / 2).attr("y", height - 18)
      .attr("text-anchor", "middle").attr("font-size", 18)
      .attr("fill", "white")
      .text("Time of Year");

    svg.append("text")
      .attr("x", -height / 2).attr("y", 40)
      .attr("transform", "rotate(-90)").attr("text-anchor", "middle")
      .attr("font-size", 18)
      .attr("fill", "white")
      .text("Total Delay Minutes");

    // Lines
    svg.append("g")
      .selectAll("path")
      .data(series)
      .join("path")
      .attr("fill", "none")
      .attr("stroke", d => d.color)
      .attr("stroke-width", 1.5)   // thinner, uniform lines
      .attr("d", d => line(d.values));

    // Add circles for each series
    series.forEach(s => {
    svg.append("g")
        .selectAll("circle")
        .data(s.values)
        .join("circle")
        .attr("cx", d => x(d.month))
        .attr("cy", d => y(d.value))
        .attr("r", 4)                // radius of points
        .attr("fill", s.color)
        .attr("stroke", "none")
        .attr("opacity", 0.8);
    });

    // Legend
    const legend = svg.append("g")
      .attr("transform", `translate(${width - margin.right + 20}, ${margin.top})`);
    legend.selectAll("legend-item")
      .data(series)
      .join("g")
      .attr("transform", (d, i) => `translate(0, ${i * 30})`)
      .each(function(d) {
        d3.select(this).append("line")
          .attr("x1", 0).attr("y1", 0).attr("x2", 28).attr("y2", 0)
          .attr("stroke", d.color).attr("stroke-width", d.strokeWidth);
        d3.select(this).append("text")
          .attr("x", 40).attr("y", 6)
          .attr("font-size", 15).attr("fill", d.color)
          .text(d.name);
      });

    // Hover interaction (same as previous)
    const hover = svg.append("g").style("pointer-events", "none").attr("opacity", 0);
    const vline = hover.append("line")
      .attr("y1", margin.top).attr("y2", height - margin.bottom)
      .attr("stroke", "white").attr("stroke-width", 1).attr("stroke-dasharray", "4,4");
    const tip = hover.append("g");
    const tipRect = tip.append("rect")
      .attr("fill", "white").attr("stroke", "#333")
      .attr("rx", 8).attr("ry", 8).attr("opacity", 0.96);
    const tipG = tip.append("g").attr("transform", "translate(12, 18)");

    const fmtInt = d3.format(",");
    const fmtPct = d3.format(".1f");

    function nearestMonth(mx) {
      const dom = x.domain();
      let best = dom[0], bestDist = Infinity;
      for (const m of dom) {
        const dist = Math.abs(x(m) - mx);
        if (dist < bestDist) { bestDist = dist; best = m; }
      }
      return best;
    }

    function showAt(event) {
      const [mx, my] = d3.pointer(event, svg.node());
      if (mx < margin.left || mx > width - margin.right || my < margin.top || my > height - margin.bottom) {
        hover.attr("opacity", 0);
        return;
      }
      const m = nearestMonth(mx);
      const row = data.find(d => d.month === m);
      if (!row) return;

      const total = row.totalDelay;
      const flights = row.totalFlights;

      const parts = [
        { label: "Carrier", val: row.carrierDelay },
        { label: "Weather", val: row.weatherDelay },
        { label: "NAS", val: row.nasDelay },
        { label: "Late Aircraft", val: row.lateAircraftDelay },
        { label: "Security", val: row.securityDelay }
      ];

      vline.attr("x1", x(m)).attr("x2", x(m));
      tipG.selectAll("*").remove();

      const lineH = 18;
      const colLabelX = 0, colMinX = 250, colPctX = 340;
      let yCursor = 0;

      tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
        .attr("font-size", 14).attr("font-weight", 800)
        .text(`${months[m - 1]} (Aggregated)`);
      yCursor += lineH + 4;

      tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
        .attr("font-size", 13).text("Total delay:");
      tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
        .attr("font-size", 13).text(`${fmtInt(Math.round(total))} min`);
      yCursor += lineH;

      tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
        .attr("font-size", 13).text("Flights:");
      tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
        .attr("font-size", 13).text(fmtInt(Math.round(flights)));
      yCursor += lineH + 10;

      tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
        .attr("font-size", 13).attr("font-weight", 800)
        .text("Delay Cause Breakdown");
      yCursor += lineH;

      tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
        .attr("font-size", 12.5).attr("font-weight", 700).text("Cause");
      tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
        .attr("font-size", 12.5).attr("font-weight", 700).text("Minutes");
      tipG.append("text").attr("x", colPctX).attr("y", yCursor).attr("text-anchor", "end")
        .attr("font-size", 12.5).attr("font-weight", 700).text("(%)");
      yCursor += lineH;

      parts.forEach(p => {
        const pct = total > 0 ? (p.val / total) * 100 : 0;
        tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
          .attr("font-size", 13).text(`${p.label}:`);
        tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
          .attr("font-size", 13).text(fmtInt(Math.round(p.val)));
        tipG.append("text").attr("x", colPctX).attr("y", yCursor).attr("text-anchor", "end")
          .attr("font-size", 13).text(`(${fmtPct(pct)}%)`);
        yCursor += lineH;
      });

      const bbox = tipG.node().getBBox();
      tipRect.attr("x", 0).attr("y", 0)
        .attr("width", bbox.width + 24).attr("height", bbox.height + 24);

      let tx = x(m) + 14, ty = my - (bbox.height + 24) / 2;
      const pad = 12;
      if (tx + (bbox.width + 24) > width - pad) tx = x(m) - (bbox.width + 24) - 14;
      if (ty < pad) ty = pad;
      if (ty + (bbox.height + 24) > height - pad) ty = height - pad - (bbox.height + 24);
      tip.attr("transform", `translate(${tx},${ty})`);
      hover.attr("opacity", 1);
    }

    svg.append("rect")
      .attr("x", margin.left).attr("y", margin.top)
      .attr("width", (width - margin.right) - margin.left)
      .attr("height", (height - margin.bottom) - margin.top)
      .attr("fill", "transparent")
      .on("mousemove", showAt)
      .on("mouseenter", () => hover.attr("opacity", 1))
      .on("mouseleave", () => hover.attr("opacity", 0));

    return svg.node();
  }

  display(resize((width) => monthlyAggChart(width)));
}
```

---

## Analysis Across Full Data Range

A similar line chart constructed for the entire date range reinforces the conclusion that delays follow an yearly pattern where delays spike during summer months, then fall until another less significant spike during December. External factors's effects, such as that of the 2008 recession or the 2020 COVID-19 pandemic can also be seen as total flight volume plummeted, causing delays to also plummet. Overall, it can be see that the aviation industry has expanded over time, which is clear by the general upward trend of peak delays each June-July.

```js
// Full monthly time series (2003–2025) with all delay causes
import * as d3 from "npm:d3";
import {resize, html} from "npm:@observablehq/stdlib";

const flightData = await FileAttachment("data/flights.csv").csv({typed: true});

// Aggregate by year + month
const monthlyMap = new Map();
for (const row of flightData) {
  const year = +row.year;
  const month = +row.month;
  const key = `${year}-${month}`;
  if (!monthlyMap.has(key)) {
    monthlyMap.set(key, {
      year, month,
      carrierDelay: 0,
      weatherDelay: 0,
      nasDelay: 0,
      securityDelay: 0,
      lateAircraftDelay: 0,
      totalFlights: 0
    });
  }
  const agg = monthlyMap.get(key);
  agg.carrierDelay += +row.carrier_delay || 0;
  agg.weatherDelay += +row.weather_delay || 0;
  agg.nasDelay += +row.nas_delay || 0;
  agg.securityDelay += +row.security_delay || 0;
  agg.lateAircraftDelay += +row.late_aircraft_delay || 0;
  agg.totalFlights += +row.arr_flights || 0;
}

let monthlyData = Array.from(monthlyMap.values());
for (const d of monthlyData) {
  d.totalDelay = d.carrierDelay + d.weatherDelay + d.nasDelay + d.securityDelay + d.lateAircraftDelay;
  d.date = new Date(d.year, d.month - 1, 1);
}
monthlyData.sort((a,b) => a.date - b.date);

function fullTimeSeriesChart(width) {
  const height = 560;
  const margin = { top: 80, right: 160, bottom: 100, left: 80 };

  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .style("overflow", "visible");

  const x = d3.scaleTime()
    .domain(d3.extent(monthlyData, d => d.date))
    .range([margin.left, width - margin.right]);

  const y = d3.scaleLinear()
    .domain([0, 16_000_000])
    .range([height - margin.bottom, margin.top]);

  // Line generator
  const line = d3.line()
    .curve(d3.curveLinear)
    .x(d => x(d.date))
    .y(d => y(d.value));

  // Define series (same colors as previous charts)
  const series = [
    { key: "total", name: "Total", color: "#2f6b1f", values: monthlyData.map(d => ({ date: d.date, value: d.totalDelay })) },
    { key: "carrierDelay", name: "Carrier Delays", color: "#7b3294", values: monthlyData.map(d => ({ date: d.date, value: d.carrierDelay })) },
    { key: "weatherDelay", name: "Weather", color: "#e31a1c", values: monthlyData.map(d => ({ date: d.date, value: d.weatherDelay })) },
    { key: "nasDelay", name: "Traffic (NAS) Delays", color: "#f39c12", values: monthlyData.map(d => ({ date: d.date, value: d.nasDelay })) },
    { key: "lateAircraftDelay", name: "Late Aircraft", color: "#1f78b4", values: monthlyData.map(d => ({ date: d.date, value: d.lateAircraftDelay })) },
    { key: "securityDelay", name: "Security", color: "#c51b7d", values: monthlyData.map(d => ({ date: d.date, value: d.securityDelay })) }
  ];

  // Draw lines for all series
  svg.append("g")
    .selectAll("path")
    .data(series)
    .join("path")
    .attr("fill", "none")
    .attr("stroke", d => d.color)
    .attr("stroke-width", 1.5)
    .attr("d", d => line(d.values));

  // Draw circles only for total delay (small, radius 2.5)
  svg.append("g")
    .selectAll("circle")
    .data(monthlyData)
    .join("circle")
    .attr("cx", d => x(d.date))
    .attr("cy", d => y(d.totalDelay))
    .attr("r", 2.5)
    .attr("fill", "#2f6b1f")
    .attr("stroke", "none")
    .attr("opacity", 0.6);

  // X axis (years, angled right/down)
  const xAxis = d3.axisBottom(x)
    .tickValues(monthlyData.filter(d => d.month === 1).map(d => d.date))
    .tickFormat(d3.timeFormat("%Y"));

  const xAxisG = svg.append("g")
    .attr("transform", `translate(0, ${height - margin.bottom})`)
    .call(xAxis);

  xAxisG.selectAll("text")
    .attr("transform", "rotate(30)")
    .attr("dx", "0.2em")
    .attr("dy", "0.6em")
    .style("text-anchor", "start")
    .attr("fill", "white")
    .attr("font-size", 12);

  // Y axis (abbreviated)
  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).tickFormat(d => (d / 1e6).toFixed(0) + "m"))
    .selectAll("text")
    .attr("fill", "white")
    .attr("font-size", 14);

  // Axis labels
  svg.append("text")
    .attr("x", width / 2).attr("y", height - 15)
    .attr("text-anchor", "middle").attr("font-size", 18)
    .attr("fill", "white")
    .text("Year");

  svg.append("text")
    .attr("x", -height / 2).attr("y", 25)
    .attr("transform", "rotate(-90)").attr("text-anchor", "middle")
    .attr("font-size", 18).attr("fill", "white")
    .text("Total Delay Minutes");

  // Title
  svg.append("text")
    .attr("x", width / 2).attr("y", 35)
    .attr("text-anchor", "middle")
    .attr("font-size", 24).attr("font-weight", 700)
    .attr("fill", "white")
    .text("Monthly Delay Minutes by Cause (June 2003 – November 2025)");

  // Legend (placed on the right)
  const legend = svg.append("g")
    .attr("transform", `translate(${width - margin.right + 10}, ${margin.top + 20})`);

  series.forEach((s, i) => {
    const g = legend.append("g").attr("transform", `translate(0, ${i * 25})`);
    g.append("line")
      .attr("x1", 0).attr("y1", 0).attr("x2", 20).attr("y2", 0)
      .attr("stroke", s.color).attr("stroke-width", 3);
    g.append("text")
      .attr("x", 28).attr("y", 4)
      .attr("font-size", 14).attr("fill", s.color)
      .text(s.name);
  });

  // ----- Tooltip (same as before, but only for total delay? Keep full breakdown) -----
  const hover = svg.append("g").style("pointer-events", "none").attr("opacity", 0);
  const vline = hover.append("line")
    .attr("y1", margin.top).attr("y2", height - margin.bottom)
    .attr("stroke", "white").attr("stroke-width", 1).attr("stroke-dasharray", "4,4");
  const tip = hover.append("g");
  const tipRect = tip.append("rect")
    .attr("fill", "white").attr("stroke", "#333")
    .attr("rx", 8).attr("ry", 8).attr("opacity", 0.96);
  const tipG = tip.append("g").attr("transform", "translate(12, 18)");

  const fmtInt = d3.format(",");
  const bisectDate = d3.bisector(d => d.date).left;

  function showTooltip(event) {
    const [mx, my] = d3.pointer(event, svg.node());
    if (mx < margin.left || mx > width - margin.right || my < margin.top || my > height - margin.bottom) {
      hover.attr("opacity", 0);
      return;
    }
    const x0 = x.invert(mx);
    const idx = bisectDate(monthlyData, x0, 1);
    const d0 = monthlyData[idx - 1];
    const d1 = monthlyData[idx];
    const row = !d1 ? d0 : (x0 - d0.date < d1.date - x0 ? d0 : d1);
    if (!row) return;

    vline.attr("x1", x(row.date)).attr("x2", x(row.date));

    const monthNames = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"];
    const header = `${monthNames[row.month-1]} ${row.year}`;
    const total = row.totalDelay;
    const flights = row.totalFlights;

    const parts = [
      { label: "Carrier", val: row.carrierDelay },
      { label: "Weather", val: row.weatherDelay },
      { label: "NAS", val: row.nasDelay },
      { label: "Late Aircraft", val: row.lateAircraftDelay },
      { label: "Security", val: row.securityDelay }
    ];

    tipG.selectAll("*").remove();
    const lineH = 18;
    const colLabelX = 0, colMinX = 250, colPctX = 340;
    let yCursor = 0;

    tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
      .attr("font-size", 14).attr("font-weight", 800)
      .text(header);
    yCursor += lineH + 4;

    tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
      .attr("font-size", 13).text("Total delay:");
    tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
      .attr("font-size", 13).text(`${fmtInt(Math.round(total))} min`);
    yCursor += lineH;

    tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
      .attr("font-size", 13).text("Flights:");
    tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
      .attr("font-size", 13).text(fmtInt(Math.round(flights)));
    yCursor += lineH + 10;

    tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
      .attr("font-size", 13).attr("font-weight", 800)
      .text("Delay Cause Breakdown");
    yCursor += lineH;

    tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
      .attr("font-size", 12.5).attr("font-weight", 700).text("Cause");
    tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
      .attr("font-size", 12.5).attr("font-weight", 700).text("Minutes");
    tipG.append("text").attr("x", colPctX).attr("y", yCursor).attr("text-anchor", "end")
      .attr("font-size", 12.5).attr("font-weight", 700).text("(%)");
    yCursor += lineH;

    parts.forEach(p => {
      const pct = total > 0 ? (p.val / total) * 100 : 0;
      tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
        .attr("font-size", 13).text(`${p.label}:`);
      tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
        .attr("font-size", 13).text(fmtInt(Math.round(p.val)));
      tipG.append("text").attr("x", colPctX).attr("y", yCursor).attr("text-anchor", "end")
        .attr("font-size", 13).text(`(${pct.toFixed(1)}%)`);
      yCursor += lineH;
    });

    const bbox = tipG.node().getBBox();
    tipRect.attr("x", 0).attr("y", 0)
      .attr("width", bbox.width + 24).attr("height", bbox.height + 24);

    let tx = x(row.date) + 14, ty = my - (bbox.height + 24) / 2;
    const pad = 12;
    if (tx + (bbox.width + 24) > width - pad) tx = x(row.date) - (bbox.width + 24) - 14;
    if (ty < pad) ty = pad;
    if (ty + (bbox.height + 24) > height - pad) ty = height - pad - (bbox.height + 24);
    tip.attr("transform", `translate(${tx},${ty})`);
    hover.attr("opacity", 1);
  }

  svg.append("rect")
    .attr("x", margin.left).attr("y", margin.top)
    .attr("width", (width - margin.right) - margin.left)
    .attr("height", (height - margin.bottom) - margin.top)
    .attr("fill", "transparent")
    .on("mousemove", showTooltip)
    .on("mouseleave", () => hover.attr("opacity", 0));

  return svg.node();
}

display(resize((width) => fullTimeSeriesChart(width))); 
```

---

## Analysis by Year (June 2003 - November 2025)

A final set of visualizations allows examination of the dataset by specific year via a dropdown. Notably, the drop in delays as a result of travel restrictions caused by the COVID-19 pandemic can be observed in 2020.

```js
// Yearly dropdown + multi‑year line chart
import * as d3 from "npm:d3";
import {resize, html} from "npm:@observablehq/stdlib";

const flightData = await FileAttachment("data/flights.csv").csv({typed: true});

// 1. Aggregate by year + month
const yearMonthMap = new Map(); // key "year-month", value: delay sums & flights
for (const row of flightData) {
  const year = +row.year;
  const month = +row.month;
  const key = `${year}-${month}`;
  if (!yearMonthMap.has(key)) {
    yearMonthMap.set(key, {
      year, month,
      carrierDelay: 0,
      weatherDelay: 0,
      nasDelay: 0,
      securityDelay: 0,
      lateAircraftDelay: 0,
      totalFlights: 0
    });
  }
  const agg = yearMonthMap.get(key);
  agg.carrierDelay += +row.carrier_delay || 0;
  agg.weatherDelay += +row.weather_delay || 0;
  agg.nasDelay += +row.nas_delay || 0;
  agg.securityDelay += +row.security_delay || 0;
  agg.lateAircraftDelay += +row.late_aircraft_delay || 0;
  agg.totalFlights += +row.arr_flights || 0;
}

// Convert to array and group by year
const allData = Array.from(yearMonthMap.values());
const years = [...new Set(allData.map(d => d.year))].sort((a,b) => a - b);

// For each year, create a 12‑month array (missing months = zeros)
const yearlyData = new Map(); // year -> array[1..12] of {month, carrierDelay, ...}
for (const y of years) {
  const monthsArr = [];
  for (let m = 1; m <= 12; m++) {
    const row = allData.find(d => d.year === y && d.month === m);
    if (row) {
      monthsArr.push({
        month: m,
        carrierDelay: row.carrierDelay,
        weatherDelay: row.weatherDelay,
        nasDelay: row.nasDelay,
        securityDelay: row.securityDelay,
        lateAircraftDelay: row.lateAircraftDelay,
        totalFlights: row.totalFlights,
        totalDelay: row.carrierDelay + row.weatherDelay + row.nasDelay + row.securityDelay + row.lateAircraftDelay
      });
    } else {
      monthsArr.push({
        month: m,
        carrierDelay: 0, weatherDelay: 0, nasDelay: 0,
        securityDelay: 0, lateAircraftDelay: 0,
        totalFlights: 0, totalDelay: 0
      });
    }
  }
  yearlyData.set(y, monthsArr);
}

// Also compute “All years” (sum across years for each month)
const allYearsData = [];
for (let m = 1; m <= 12; m++) {
  let carrier = 0, weather = 0, nas = 0, security = 0, late = 0, flights = 0;
  for (const y of years) {
    const monthData = yearlyData.get(y)[m-1];
    carrier += monthData.carrierDelay;
    weather += monthData.weatherDelay;
    nas += monthData.nasDelay;
    security += monthData.securityDelay;
    late += monthData.lateAircraftDelay;
    flights += monthData.totalFlights;
  }
  allYearsData.push({
    month: m,
    carrierDelay: carrier,
    weatherDelay: weather,
    nasDelay: nas,
    securityDelay: security,
    lateAircraftDelay: late,
    totalFlights: flights,
    totalDelay: carrier + weather + nas + security + late
  });
}

// ----- Chart drawing function (identical to previous, but accepts 'monthlyData' array and a title suffix) -----
function drawMonthChart(monthlyData, titleSuffix, width) {
  const height = 560;
  const margin = { top: 80, right: 230, bottom: 80, left: 125 };
  const months = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"];

  // Ensure dailyData has 12 entries (it should, but safety)
  const data = monthlyData.slice(); // copy

  const x = d3.scalePoint()
    .domain(data.map(d => d.month))
    .range([margin.left, width - margin.right])
    .padding(0.25);

  const maxY = d3.max(data, d => Math.max(d.totalDelay, d.carrierDelay, d.weatherDelay, d.nasDelay, d.securityDelay, d.lateAircraftDelay));
  const y = d3.scaleLinear()
    .domain([0, 16_000_000])
    .range([height - margin.bottom, margin.top]);

  const line = d3.line()
    .curve(d3.curveLinear)
    .x(d => x(d.month))
    .y(d => y(d.value));

  const series = [
    { key: "total", name: "Total", color: "#2f6b1f",
      values: data.map(d => ({ month: d.month, value: d.totalDelay })) },
    { key: "carrierDelay", name: "Carrier Delays", color: "#7b3294",
      values: data.map(d => ({ month: d.month, value: d.carrierDelay })) },
    { key: "weatherDelay", name: "Weather", color: "#e31a1c",
      values: data.map(d => ({ month: d.month, value: d.weatherDelay })) },
    { key: "nasDelay", name: "Traffic (NAS) Delays", color: "#f39c12",
      values: data.map(d => ({ month: d.month, value: d.nasDelay })) },
    { key: "lateAircraftDelay", name: "Late Aircraft", color: "#1f78b4",
      values: data.map(d => ({ month: d.month, value: d.lateAircraftDelay })) },
    { key: "securityDelay", name: "Security", color: "#c51b7d",
      values: data.map(d => ({ month: d.month, value: d.securityDelay })) }
  ];

  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .style("overflow", "visible");

  // Title (with dynamic suffix)
  svg.append("text")
    .attr("x", width / 2).attr("y", 40)
    .attr("text-anchor", "middle")
    .attr("font-size", 24).attr("font-weight", 700)
    .attr("fill", "white")
    .text(`Monthly Airline Delay Minutes by Cause (${titleSuffix})`);

  // Axes
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(x).tickFormat(m => months[m - 1]))
    .selectAll("text").attr("font-size", 14).attr("fill", "white");

  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).tickFormat(d => (d / 1e6).toFixed(0) + "m"))
    .selectAll("text")
    .attr("font-size", 14)
    .attr("fill", "white");

  // Axis labels
  svg.append("text")
    .attr("x", width / 2).attr("y", height - 18)
    .attr("text-anchor", "middle").attr("font-size", 18)
    .attr("fill", "white")
    .text("Time of Year");

  svg.append("text")
    .attr("x", -height / 2).attr("y", 40)
    .attr("transform", "rotate(-90)").attr("text-anchor", "middle")
    .attr("font-size", 18)
    .attr("fill", "white")
    .text("Total Delay Minutes");

  // Lines (thin, uniform)
  svg.append("g")
    .selectAll("path")
    .data(series)
    .join("path")
    .attr("fill", "none")
    .attr("stroke", d => d.color)
    .attr("stroke-width", 1.5)
    .attr("d", d => line(d.values));

  // Circles
  series.forEach(s => {
    svg.append("g")
      .selectAll("circle")
      .data(s.values)
      .join("circle")
      .attr("cx", d => x(d.month))
      .attr("cy", d => y(d.value))
      .attr("r", 4)
      .attr("fill", s.color)
      .attr("stroke", "none")
      .attr("opacity", 0.8);
  });

  // Legend
  const legend = svg.append("g")
    .attr("transform", `translate(${width - margin.right + 20}, ${margin.top})`);
  legend.selectAll("legend-item")
    .data(series)
    .join("g")
    .attr("transform", (d, i) => `translate(0, ${i * 30})`)
    .each(function(d) {
      d3.select(this).append("line")
        .attr("x1", 0).attr("y1", 0).attr("x2", 28).attr("y2", 0)
        .attr("stroke", d.color).attr("stroke-width", 3);
      d3.select(this).append("text")
        .attr("x", 40).attr("y", 6)
        .attr("font-size", 15).attr("fill", d.color)
        .text(d.name);
    });

  // --------------------------
  // Hover interaction (vertical line + tooltip)
  // --------------------------
  const hover = svg.append("g").style("pointer-events", "none").attr("opacity", 0);
  const vline = hover.append("line")
    .attr("y1", margin.top).attr("y2", height - margin.bottom)
    .attr("stroke", "white").attr("stroke-width", 1).attr("stroke-dasharray", "4,4");
  const tip = hover.append("g");
  const tipRect = tip.append("rect")
    .attr("fill", "white").attr("stroke", "#333")
    .attr("rx", 8).attr("ry", 8).attr("opacity", 0.96);
  const tipG = tip.append("g").attr("transform", "translate(12, 18)");

  const fmtInt = d3.format(",");
  const fmtPct = d3.format(".1f");

  function nearestMonth(mx) {
    const dom = x.domain();
    let best = dom[0], bestDist = Infinity;
    for (const m of dom) {
      const dist = Math.abs(x(m) - mx);
      if (dist < bestDist) { bestDist = dist; best = m; }
    }
    return best;
  }

  function showAt(event) {
    const [mx, my] = d3.pointer(event, svg.node());
    if (mx < margin.left || mx > width - margin.right || my < margin.top || my > height - margin.bottom) {
      hover.attr("opacity", 0);
      return;
    }
    const m = nearestMonth(mx);
    const row = data.find(d => d.month === m);
    if (!row) return;

    const total = row.totalDelay;
    const flights = row.totalFlights;

    const parts = [
      { label: "Carrier", val: row.carrierDelay },
      { label: "Weather", val: row.weatherDelay },
      { label: "NAS", val: row.nasDelay },
      { label: "Late Aircraft", val: row.lateAircraftDelay },
      { label: "Security", val: row.securityDelay }
    ];

    vline.attr("x1", x(m)).attr("x2", x(m));
    tipG.selectAll("*").remove();

    const lineH = 18;
    const colLabelX = 0, colMinX = 250, colPctX = 340;
    let yCursor = 0;

    tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
      .attr("font-size", 14).attr("font-weight", 800)
      .text(`${months[m - 1]} (${titleSuffix})`);
    yCursor += lineH + 4;

    tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
      .attr("font-size", 13).text("Total delay:");
    tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
      .attr("font-size", 13).text(`${fmtInt(Math.round(total))} min`);
    yCursor += lineH;

    tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
      .attr("font-size", 13).text("Flights:");
    tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
      .attr("font-size", 13).text(fmtInt(Math.round(flights)));
    yCursor += lineH + 10;

    tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
      .attr("font-size", 13).attr("font-weight", 800)
      .text("Delay Cause Breakdown");
    yCursor += lineH;

    tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
      .attr("font-size", 12.5).attr("font-weight", 700).text("Cause");
    tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
      .attr("font-size", 12.5).attr("font-weight", 700).text("Minutes");
    tipG.append("text").attr("x", colPctX).attr("y", yCursor).attr("text-anchor", "end")
      .attr("font-size", 12.5).attr("font-weight", 700).text("(%)");
    yCursor += lineH;

    parts.forEach(p => {
      const pct = total > 0 ? (p.val / total) * 100 : 0;
      tipG.append("text").attr("x", colLabelX).attr("y", yCursor)
        .attr("font-size", 13).text(`${p.label}:`);
      tipG.append("text").attr("x", colMinX).attr("y", yCursor).attr("text-anchor", "end")
        .attr("font-size", 13).text(fmtInt(Math.round(p.val)));
      tipG.append("text").attr("x", colPctX).attr("y", yCursor).attr("text-anchor", "end")
        .attr("font-size", 13).text(`(${fmtPct(pct)}%)`);
      yCursor += lineH;
    });

    const bbox = tipG.node().getBBox();
    tipRect.attr("x", 0).attr("y", 0)
      .attr("width", bbox.width + 24).attr("height", bbox.height + 24);

    let tx = x(m) + 14, ty = my - (bbox.height + 24) / 2;
    const pad = 12;
    if (tx + (bbox.width + 24) > width - pad) tx = x(m) - (bbox.width + 24) - 14;
    if (ty < pad) ty = pad;
    if (ty + (bbox.height + 24) > height - pad) ty = height - pad - (bbox.height + 24);
    tip.attr("transform", `translate(${tx},${ty})`);
    hover.attr("opacity", 1);
  }

  svg.append("rect")
    .attr("x", margin.left).attr("y", margin.top)
    .attr("width", (width - margin.right) - margin.left)
    .attr("height", (height - margin.bottom) - margin.top)
    .attr("fill", "transparent")
    .on("mousemove", showAt)
    .on("mouseenter", () => hover.attr("opacity", 1))
    .on("mouseleave", () => hover.attr("opacity", 0));

  return svg.node();
}

// ----- UI: dropdown + chart container -----
const container = document.createElement("div");

// Dropdown selector
const select = document.createElement("select");
select.style.marginBottom = "20px";
select.style.padding = "8px";
select.style.fontSize = "16px";
select.style.backgroundColor = "#222";
select.style.color = "white";
select.style.border = "1px solid #555";

// const allOption = document.createElement("option");
// allOption.value = "all";
// allOption.textContent = "All years (aggregated)";
// select.appendChild(allOption);
for (const y of years) {
  const opt = document.createElement("option");
  opt.value = y;
  opt.textContent = y;
  select.appendChild(opt);
}
container.appendChild(select);

// Chart mount point
const chartDiv = document.createElement("div");
container.appendChild(chartDiv);

// Helper to update chart based on selection
// Replace the updateChart function with this version
let currentWrapper = null;

function updateChart() {
  const selected = select.value;
  let monthlyData, titleSuffix;
  const year = +selected;
  monthlyData = yearlyData.get(year);
  titleSuffix = `${year}`;
  if (!monthlyData) return;

  const containerWidth = chartDiv.clientWidth || 900;
  const newSvg = drawMonthChart(monthlyData, titleSuffix, containerWidth);
  newSvg.classList.add("chart-svg");
  newSvg.style.opacity = "0";

  if (currentWrapper) {
    // Fade out current
    currentWrapper.style.transition = "opacity 0.2s ease";
    currentWrapper.style.opacity = "0";
    setTimeout(() => {
      chartDiv.removeChild(currentWrapper);
      chartDiv.appendChild(newSvg);
      // Force reflow (multiple methods)
      void newSvg.offsetHeight;                // forces layout
      window.getComputedStyle(newSvg).opacity; // also forces reflow
      // Then fade in
      newSvg.style.transition = "opacity 0.2s ease";
      newSvg.style.opacity = "1";
      currentWrapper = newSvg;
    }, 200);
  } else {
    chartDiv.appendChild(newSvg);
    void newSvg.offsetHeight;
    newSvg.style.transition = "opacity 0.2s ease";
    newSvg.style.opacity = "1";
    currentWrapper = newSvg;
  }
}
select.addEventListener("change", () => updateChart());

// Initial draw
updateChart();

display(container);
```




