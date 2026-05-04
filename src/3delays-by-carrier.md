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

# Flight Delays by Airline

---

## Carrier-based Analysis

To understand how airlines differ in their delay performance at the carrier levevl, a bubble scatterplot was created to compare two metrics: the percentage of flights delayed and the average severity of those delays. Each bubble represents an airline, positioned by its delay rate on the horizontal axis and average delay minutes on the vertical axis. The bubble size and color intensity encode the total flight volume for that airline, with darker red indicating higher volume. The chart shows that airlines cluster into distinct groups: some have low delay rates but moderate delay severity, while others have high delay rates paired with longer average delays. 

The tooltip provides three metrics for each airline: the percentage of flights that experienced delays, the average delay duration in minutes, and the total number of flights operated over the period. This structure allows for comparison of airline performance independent of volume, showing which carriers are more efficient at moving passengers despite differences in carriers' scale.

```js
// Carrier bubble plot (all years 2003–2025)
import * as d3 from "npm:d3";
import {resize, html} from "npm:@observablehq/stdlib";

const flightData = await FileAttachment("data/flights.csv").csv({typed: true});

// Get year range for title
const minYear = d3.min(flightData, d => +d.year);
const maxYear = d3.max(flightData, d => +d.year);

// Aggregate all years by carrier_name
const carrierMap = new Map(); // carrier_name -> { flights, delayedFlights, totalDelay }

for (const row of flightData) {
  const carrier = row.carrier_name?.trim();
  if (!carrier) continue;
  const flights = +row.arr_flights || 0;
  const delayed = +row.arr_del15 || 0;
  const delayMins = +row.arr_delay || 0;
  const existing = carrierMap.get(carrier);
  if (existing) {
    existing.flights += flights;
    existing.delayedFlights += delayed;
    existing.totalDelay += delayMins;
  } else {
    carrierMap.set(carrier, { flights, delayedFlights: delayed, totalDelay: delayMins });
  }
}

// Convert to array and compute derived metrics
const data = Array.from(carrierMap.entries()).map(([carrier, vals]) => {
  const pctDelayed = vals.flights > 0 ? vals.delayedFlights / vals.flights : 0;
  const avgDelayMinutes = vals.flights > 0 ? vals.totalDelay / vals.flights : 0;
  return { carrier, flights: vals.flights, pctDelayed, avgDelayMinutes };
});

if (data.length === 0) {
  display(html`<div style="color:red;">No carrier data found.</div>`);
} else {
  function carrierBubblePlot(width) {
    const height = 800;
    const margin = { top: 70, right: 40, bottom: 300, left: 90 };

    const svg = d3.create("svg")
      .attr("width", width)
      .attr("height", height)
      .style("overflow", "visible");

    // Scales
    const x = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.pctDelayed) * 1.1])
      .range([margin.left, width - margin.right]);

    const y = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.avgDelayMinutes) * 1.1])
      .range([height - margin.bottom, margin.top]);

    // Sequential color: lighter = fewer flights, darker = more flights
    const color = d3.scaleSequential()
      .domain(d3.extent(data, d => d.flights))
      .interpolator(t => d3.interpolateRgb("#ffffff", "#ff0000")(t));

    // Title
    svg.append("text")
      .attr("x", width / 2).attr("y", 34)
      .attr("text-anchor", "middle")
      .attr("font-size", 22).attr("font-weight", 700)
      .attr("fill", "white")
      .text(`Airline Delay Rate vs Average Delay Severity (${minYear}–${maxYear})`);

    // Gridlines
    svg.append("g").attr("stroke", "#555")
      .selectAll("line").data(d3.range(margin.left, width - margin.right, 50))
      .join("line").attr("x1", d => d).attr("x2", d => d)
      .attr("y1", margin.top).attr("y2", height - margin.bottom);

    svg.append("g").attr("stroke", "#555")
      .selectAll("line-h").data(d3.range(margin.top, height - margin.bottom, 50))
      .join("line").attr("y1", d => d).attr("y2", d => d)
      .attr("x1", margin.left).attr("x2", width - margin.right);

    // Bubbles
    svg.append("g")
      .selectAll("circle")
      .data(data)
      .join("circle")
      .attr("cx", d => x(d.pctDelayed))
      .attr("cy", d => y(d.avgDelayMinutes))
      .attr("r", 6)
      .attr("fill", d => color(d.flights))
      .attr("fill-opacity", 0.8)
      .attr("stroke", "#333").attr("stroke-width", 1)
      .on("mousemove", (event, d) => showTooltip(d, event))
      .on("mouseleave", hideTooltip);

    // Tooltip
    const tooltip = svg.append("g").style("pointer-events", "none").attr("opacity", 0);
    const tipRect = tooltip.append("rect")
      .attr("fill", "white").attr("stroke", "#333").attr("rx", 5).attr("ry", 5);
    const tipText = tooltip.append("text").attr("font-size", 14).attr("x", 8).attr("y", 18).attr("fill", "#222");

    function showTooltip(d, event) {
      const lines = [
        d.carrier,
        `${(d.pctDelayed * 100).toFixed(1)}% flights delayed`,
        `Avg delay: ${d.avgDelayMinutes.toFixed(1)} min`,
        `${d.flights.toLocaleString()} flights`
      ];
      tipText.selectAll("tspan").remove();
      lines.forEach((line, i) =>
        tipText.append("tspan").attr("x", 8).attr("dy", i === 0 ? 0 : 16).text(line)
      );
      const bbox = tipText.node().getBBox();
      tipRect.attr("width", bbox.width + 16).attr("height", bbox.height + 12);
      const [mx, my] = d3.pointer(event, svg.node());
      tooltip.attr("transform", `translate(${mx - bbox.width - 40}, ${my - 10})`).attr("opacity", 1);
    }

    function hideTooltip() { tooltip.attr("opacity", 0); }

    // Axes
    svg.append("g")
      .attr("transform", `translate(0, ${height - margin.bottom})`)
      .call(d3.axisBottom(x).tickFormat(d => `${Math.round(d * 100)}%`))
      .selectAll("text").attr("fill", "white").attr("font-size", 14);

    svg.append("g")
      .attr("transform", `translate(${margin.left},0)`)
      .call(d3.axisLeft(y).tickFormat(m => `${m.toFixed(0)} min`))
      .selectAll("text").attr("fill", "white").attr("font-size", 14);

    // Axis labels
    svg.append("text")
      .attr("x", width / 2).attr("y", height - 250)
      .attr("text-anchor", "middle").attr("font-size", 18).attr("fill", "white")
      .text("% of Flights Delayed");

    const plotMidY = margin.top + (height - margin.bottom - margin.top) / 2;
    svg.append("text")
      .attr("transform", `translate(28, ${plotMidY}) rotate(-90)`)
      .attr("text-anchor", "middle").attr("font-size", 18).attr("fill", "white")
      .text("Average Delay Severity (Minutes)");

    // Legend: bubble size & color
    const legendGroup = svg.append("g")
      .attr("transform", `translate(${width - 240}, ${height - margin.bottom + 60})`);

    const colorLegend = legendGroup.append("g").attr("transform", "translate(0, 0)");
    colorLegend.append("text")
      .text("Volume of Flights")
      .attr("font-size", 14).attr("fill", "white").attr("y", 20);

    const gradId = `flightColor-${Math.random()}`;
    const defs = svg.append("defs");
    const gradient = defs.append("linearGradient").attr("id", gradId).attr("x1", "0%").attr("x2", "100%");
    const [d0, d1] = color.domain();
    [0, 0.25, 0.5, 0.75, 1].forEach(t => {
      gradient.append("stop").attr("offset", `${t * 100}%`).attr("stop-color", color(d0 + t * (d1 - d0)));
    });
    colorLegend.append("rect")
      .attr("x", 0).attr("y", 25).attr("width", 200).attr("height", 14)
      .attr("fill", `url(#${gradId})`).attr("fill-opacity", 0.8).attr("stroke", "#444");

    const colorScale = d3.scaleLinear().domain(color.domain()).range([0, 200]);
    colorLegend.append("g")
      .attr("transform", "translate(0, 32)")
      .call(d3.axisBottom(colorScale).ticks(4).tickFormat(d => (d / 1e6).toFixed(0) + " mil."))
      .call(g => g.select(".domain").remove())
      .selectAll("text").attr("fill", "white").attr("font-size", 12);

    return svg.node();
  }

  display(resize((width) => carrierBubblePlot(width)));
}
```

---

## Most Delayed Carrier's Cause Breakdown

A stacked bar chart was made to display the composition of delay causes for the ten airlines with the highest total delay minutes. Each bar represents an airline and is divided into five . The chart reveals that delay cause composition varies across airlines. Some carriers, like Southwest Airlines Co., have a higher proportion of NAS/Traffic delays, while others, like American Airlines Inc., are dominated by Late Aircraft delays. This breakdown shows that the factors contributing to delays are not uniform across the industry.

The visualization supports two forms of interaction: hovering over a segment displays a tooltip showing the percentage of total delay minutes for that specific cause and the absolute delay minutes for that airline, while clicking a color in the legend highlights that cause across all airlines simultaneously. This dual interaction allows users to examine either individual airline profiles or compare how a single delay cause varies across the industry.

```js
// Stacked bar chart: Top 10 airlines by delay cause (2003–2025) with selectable legend
import * as d3 from "npm:d3";
import {resize, html} from "npm:@observablehq/stdlib";

const flightData = await FileAttachment("data/flights.csv").csv({typed: true});

// Aggregate all years by carrier
const carrierMap = new Map(); // carrier_name -> { totalDelay, causeDelays }

for (const row of flightData) {
  const carrier = row.carrier_name?.trim();
  if (!carrier) continue;
  const delay = +row.arr_delay || 0;
  const existing = carrierMap.get(carrier);
  if (existing) {
    existing.totalDelay += delay;
    existing.carrierDelay += +row.carrier_delay || 0;
    existing.weatherDelay += +row.weather_delay || 0;
    existing.nasDelay += +row.nas_delay || 0;
    existing.lateAircraftDelay += +row.late_aircraft_delay || 0;
    existing.securityDelay += +row.security_delay || 0;
  } else {
    carrierMap.set(carrier, {
      totalDelay: delay,
      carrierDelay: +row.carrier_delay || 0,
      weatherDelay: +row.weather_delay || 0,
      nasDelay: +row.nas_delay || 0,
      lateAircraftDelay: +row.late_aircraft_delay || 0,
      securityDelay: +row.security_delay || 0
    });
  }
}

// Convert to array, compute cause percentages (share of total delay)
const allCarriers = Array.from(carrierMap.entries()).map(([carrier, vals]) => {
  const total = vals.totalDelay;
  const propCarrier = total > 0 ? (vals.carrierDelay / total) * 100 : 0;
  const propWeather = total > 0 ? (vals.weatherDelay / total) * 100 : 0;
  const propNas = total > 0 ? (vals.nasDelay / total) * 100 : 0;
  const propLateAircraft = total > 0 ? (vals.lateAircraftDelay / total) * 100 : 0;
  const propSecurity = total > 0 ? (vals.securityDelay / total) * 100 : 0;
  return { carrier, totalDelay: total, propCarrier, propWeather, propNas, propLateAircraft, propSecurity };
});

// Sort by total delay descending, take top 10
const top10 = allCarriers.sort((a,b) => b.totalDelay - a.totalDelay).slice(0, 10);

// Define causes (order matters for stacking)
const causes = [
  { key: "propCarrier", label: "Carrier / Operations", color: "#4c72b0" },
  { key: "propWeather", label: "Weather", color: "#dd8452" },
  { key: "propNas", label: "NAS / Traffic", color: "#55a868" },
  { key: "propLateAircraft", label: "Late Aircraft", color: "#c44e52" },
  { key: "propSecurity", label: "Security", color: "#8172b2" }
];

if (top10.length === 0) {
  display(html`<div style="color:red;">No airline data found.</div>`);
} else {
  function topAirlineChart(width) {
    const height = 550;
    const margin = { top: 60, right: 220, bottom: 180, left: 70 }; // extra bottom space

    const svg = d3.create("svg")
      .attr("width", width)
      .attr("height", height)
      .style("overflow", "visible");

    // Scales
    const x = d3.scaleBand()
      .domain(top10.map(d => d.carrier))
      .range([margin.left, width - margin.right])
      .padding(0.2);

    const y = d3.scaleLinear()
      .domain([0, 100])
      .nice()
      .range([height - margin.bottom, margin.top]);

    const color = d3.scaleOrdinal()
      .domain(causes.map(c => c.key))
      .range(causes.map(c => c.color));

    // Stack data
    const stack = d3.stack().keys(causes.map(c => c.key));
    const stackedData = stack(top10);

    // Axes
    svg.append("g")
      .attr("transform", `translate(0, ${height - margin.bottom})`)
      .call(d3.axisBottom(x))
      .selectAll("text")
      .attr("text-anchor", "end")
      .attr("transform", "rotate(-40)")
      .attr("dx", "-0.6em").attr("dy", "0.1em")
      .attr("fill", "white")
      .attr("font-size", 12);

    svg.append("g")
      .attr("transform", `translate(${margin.left},0)`)
      .call(d3.axisLeft(y).ticks(5).tickFormat(d => d + "%"))
      .selectAll("text").attr("fill", "white");

    // Axis labels
    svg.append("text")
      .attr("x", width / 2 - 100).attr("y", height - margin.bottom + 120)
      .attr("text-anchor", "middle").attr("font-size", 15).attr("fill", "white")
      .text("Airline (Top 10 by Total Delay Minutes, 2003–2025)");

    svg.append("text")
      .attr("x", -height / 2 + 70).attr("y", 22).attr("transform", "rotate(-90)")
      .attr("text-anchor", "middle").attr("font-size", 15).attr("fill", "white")
      .text("Share of Total Delay Minutes");

    svg.append("text")
      .attr("x", width / 2 - 100).attr("y", 32).attr("text-anchor", "middle")
      .attr("font-size", 20).attr("font-weight", 700).attr("fill", "white")
      .text("Delay Cause Composition by Airline (2003–2025)");

    // Tooltip (will be raised to front later)
    const tooltip = svg.append("g")
      .style("pointer-events", "none")
      .attr("opacity", 0);
    const tipRect = tooltip.append("rect")
      .attr("fill", "white").attr("stroke", "#333").attr("rx", 4).attr("ry", 4);
    const tipText = tooltip.append("text")
      .attr("font-size", 13).attr("x", 8).attr("y", 16).attr("fill", "#222");

    let selectedCauseKey = null; // null = show all

    // Function to draw/update bars with current selection
    function drawBars() {
      const groups = svg.selectAll("g.stack-layer")
        .data(stackedData, d => d.key);
      groups.exit().remove();
      const groupsEnter = groups.enter().append("g")
        .attr("class", "stack-layer")
        .attr("fill", d => color(d.key));
      const allGroups = groupsEnter.merge(groups);

      allGroups.selectAll("rect")
        .data(d => d)
        .join("rect")
        .attr("x", d => x(d.data.carrier))
        .attr("y", d => y(d[1]))
        .attr("height", d => y(d[0]) - y(d[1]))
        .attr("width", x.bandwidth())
        .attr("opacity", function(d) {
          const layerKey = d3.select(this.parentNode).datum().key;
          return (selectedCauseKey === null || layerKey === selectedCauseKey) ? 1 : 0.15;
        });
    }

    drawBars(); // initial draw

    // Attach tooltip interaction to all bar rectangles
    svg.selectAll("g.stack-layer").selectAll("rect")
      .on("mousemove", function(event, d) {
        const layerKey = d3.select(this.parentNode).datum().key;
        let cause, value;
        if (selectedCauseKey !== null) {
          cause = causes.find(c => c.key === selectedCauseKey);
          value = d.data[selectedCauseKey]; // percentage directly from data
        } else {
          cause = causes.find(c => c.key === layerKey);
          value = d[1] - d[0]; // height of hovered segment
        }
        const lines = [
          d.data.carrier,
          `${cause.label}: ${value.toFixed(1)}% of delay minutes`,
          `Total delay: ${d.data.totalDelay.toLocaleString()} minutes`
        ];
        tipText.selectAll("tspan").remove();
        lines.forEach((line, i) =>
          tipText.append("tspan").attr("x", 8).attr("dy", i === 0 ? 0 : 16).text(line)
        );
        const bbox = tipText.node().getBBox();
        tipRect.attr("width", bbox.width + 16).attr("height", bbox.height + 12);
        const [mx, my] = d3.pointer(event, svg.node());
        tooltip.attr("transform", `translate(${mx + 12}, ${my - 10})`).attr("opacity", 1);
      })
      .on("mouseleave", () => tooltip.attr("opacity", 0));

    // Legend with click selection
    const legend = svg.append("g")
      .attr("transform", `translate(${width - margin.right + 20}, ${margin.top})`);

    causes.forEach((cause, i) => {
      const g = legend.append("g")
        .attr("transform", `translate(0, ${i * 24})`)
        .style("cursor", "pointer")
        .on("click", () => {
          if (selectedCauseKey === cause.key) {
            selectedCauseKey = null; // deselect
          } else {
            selectedCauseKey = cause.key;
          }
          drawBars(); // update opacities
          // Update legend border: remove all, then add to selected if any
          legend.selectAll("rect.legend-bg").attr("stroke", "none");
          if (selectedCauseKey !== null) {
            const idx = causes.findIndex(c => c.key === selectedCauseKey);
            legend.selectAll("g").filter((d, j) => j === idx)
              .select("rect.legend-bg").attr("stroke", "white").attr("stroke-width", 2);
          }
        });

      g.append("rect")
        .attr("width", 14).attr("height", 14)
        .attr("fill", cause.color)
        .attr("class", "legend-bg");
      g.append("text")
        .attr("x", 20).attr("y", 11)
        .attr("font-size", 13).attr("fill", "white")
        .text(cause.label);
    });

    // Ensure tooltip appears above legend
    tooltip.raise();

    return svg.node();
  }

  display(resize((width) => topAirlineChart(width)));
}
```

