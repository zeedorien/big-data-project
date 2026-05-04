---
toc: false
---

# Flight Delays by State

# Average Delay Minutes Per Flight by State, 2003-2025

The most intuitive analysis at first was to examine whether geographic location had an effect on flight delays, so a choropleth was created to display average delay per flight across states over a 22-year period. The visualization uses a color scale ranging from white (9.0 minutes) to dark red (14.5 minutes) to encode delay values.

```js
// IMPORTS
import * as d3 from "npm:d3";
import {geoAlbersUsa, geoPath} from "npm:d3-geo";
import {resize, html} from "npm:@observablehq/stdlib";
import * as topojson from "npm:topojson";

// LOAD AND AGGREGATE DATA
const data = await FileAttachment("data/flights.csv").csv({typed: true});

function extractState(airportName) {
  const match = airportName?.match(/,\s*([A-Z]{2})\s*:/);
  return match ? match[1] : null;
}

const stateMap = new Map();

for (const row of data) {
  const state = extractState(row.airport_name);
  if (!state) continue;
  const flights = +row.arr_flights || 0;
  const delay = +row.arr_delay || 0;
  const existing = stateMap.get(state);
  if (existing) {
    existing.flights += flights;
    existing.totalDelay += delay;
  } else {
    stateMap.set(state, { flights, totalDelay: delay });
  }
}

const airportTable = Array.from(stateMap.entries()).map(([state, { flights, totalDelay }]) => ({
  state,
  flights,
  totalDelay
}));

if (airportTable.length === 0) {
  display(html`<div style="color:red; padding:20px;">No state data found. Check airport_name extraction.</div>`);
} else {
  // GEOJSON
  let geojson;
  try {
    const topo = await fetch("https://cdn.jsdelivr.net/npm/us-atlas@3/states-10m.json").then(r => r.json());
    // Convert the 'states' object to GeoJSON
    geojson = topojson.feature(topo, topo.objects.states);
  } catch (err) {
    display(html`<div style="color:red; padding:20px;">Failed to load US map data: ${err.message}</div>`);
    // Stop execution by throwing? But we can just return early using a dummy div – but we are inside an else block, so we'll set a flag.
    // We'll wrap the rest in a condition.
    geojson = null;
  }

  if (geojson && geojson.features && geojson.features.length) {
    // MAP FUNCTION
    function stateDelayMap(width) {
      const height = 720;
      const legendSpace = 110;
      const mapHeight = height - legendSpace;

      const svg = d3.create("svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", `0 0 ${width} ${height}`)
        .style("overflow", "visible")
        //.style("border", "2px solid red"); // debug

      const fipsToAbbr = {
        "01":"AL","02":"AK","04":"AZ","05":"AR","06":"CA","08":"CO","09":"CT",
        "10":"DE","11":"DC","12":"FL","13":"GA","15":"HI","16":"ID","17":"IL",
        "18":"IN","19":"IA","20":"KS","21":"KY","22":"LA","23":"ME","24":"MD",
        "25":"MA","26":"MI","27":"MN","28":"MS","29":"MO","30":"MT","31":"NE",
        "32":"NV","33":"NH","34":"NJ","35":"NM","36":"NY","37":"NC","38":"ND",
        "39":"OH","40":"OK","41":"OR","42":"PA","44":"RI","45":"SC","46":"SD",
        "47":"TN","48":"TX","49":"UT","50":"VT","51":"VA","53":"WA","54":"WV",
        "55":"WI","56":"WY"
      };

      const statsByState = new Map();
      for (const row of airportTable) {
        const perFlight = row.flights > 0 ? row.totalDelay / row.flights : 0;
        statsByState.set(row.state, {
          flights: row.flights,
          delayMin: row.totalDelay,
          delayMinPerFlight: perFlight
        });
      }

      const values = Array.from(statsByState.values(), d => d.delayMinPerFlight);
      if (values.length === 0) {
        svg.append("text").attr("x", width/2).attr("y", height/2).text("No delay data").attr("text-anchor", "middle");
        return svg.node();
      }

      const color = d3.scaleSequential()
        .domain([d3.quantile(values, 0.05), d3.quantile(values, 0.95)])
        .interpolator(t => d3.interpolateRgb("white", "maroon")(t))
        .clamp(true);

      const projection = geoAlbersUsa()
        .scale(1200)
        .translate([width / 2, mapHeight / 2 + 20]);

      const path = geoPath(projection);
      const mapLayer = svg.append("g");

      mapLayer.selectAll("path")
        .data(geojson.features)
        .join("path")
          .attr("d", d => path(d))
          .attr("fill", d => {
            const abbr = fipsToAbbr[d.id];
            const s = statsByState.get(abbr);
            return s == null ? "#eee" : color(s.delayMinPerFlight);
          })
          .attr("stroke", "#555")
          .attr("stroke-width", 0.5)
          .on("mousemove", function(event, d) {
            const abbr = fipsToAbbr[d.id];
            const s = statsByState.get(abbr);
            const name = d.properties?.name || "Unknown";
            showTooltip(name, abbr, s, event);
          })
          .on("mouseleave", hideTooltip);

      // Tooltip
      const tooltip = svg.append("g").style("pointer-events", "none").attr("opacity", 0);
      const tipRect = tooltip.append("rect").attr("fill", "white").attr("stroke", "#333").attr("rx", 4).attr("ry", 4).attr("opacity", 0.95);
      const tipText = tooltip.append("text").attr("font-size", 14).attr("x", 8).attr("y", 18).attr("font-weight", 600);

      function showTooltip(name, abbr, s, event) {
        const flights = s?.flights ?? 0;
        const delayMin = s?.delayMin ?? 0;
        const perFlight = flights > 0 ? delayMin / flights : 0;
        const lines = [
          `${name} (${abbr})`,
          `${perFlight.toFixed(2)} minutes delay per flight`,
          `${delayMin.toLocaleString()} total delay minutes`,
          `${flights.toLocaleString()} total flights`
        ];
        tipText.selectAll("tspan").remove();
        lines.forEach((line, i) => {
          tipText.append("tspan").attr("x", 8).attr("dy", i === 0 ? 0 : 16).text(line);
        });
        const bbox = tipText.node().getBBox();
        tipRect.attr("width", bbox.width + 16).attr("height", bbox.height + 12);
        const [x, y] = d3.pointer(event, svg.node());
        tooltip.attr("transform", `translate(${x + 15},${y - 10})`).attr("opacity", 1);
      }

      function hideTooltip() { tooltip.attr("opacity", 0); }

      // Title
      svg.append("text")
        .attr("x", width / 2)
        .attr("y", 34)
        .attr("text-anchor", "middle")
        .attr("fill", "white")
        .attr("font-size", 24)
        .attr("font-weight", 700)
        .text("Average Delay Minutes Per Flight by State, 2003-2025");

      // Legend
      const legendWidth = 340, legendHeight = 16;
      const legendX = width - legendWidth - 80, legendY = height - 70;
      const legend = svg.append("g").attr("transform", `translate(${legendX}, ${legendY})`);
      const defs = svg.append("defs");
      const gradId = "delayGradBlue";
      const grad = defs.append("linearGradient").attr("id", gradId);
      grad.append("stop")
        .attr("offset", "0%")
        .attr("stop-color", color(color.domain()[0]));
      grad.append("stop")
        .attr("offset", "100%")
        .attr("stop-color", color(color.domain()[1]));
      legend.append("text")
        .attr("x", 0)
        .attr("y", -10)
        .attr("font-size", 16)
        .attr("font-weight", 700)
        .attr("fill", "white")
        .text("Avg delay per flight (minutes)");
      legend.append("rect")
        .attr("width", legendWidth)
        .attr("height", legendHeight)
        .attr("fill", `url(#${gradId})`)
        .attr("stroke", "#444");
      const scale = d3.scaleLinear()
        .domain(color.domain())
        .range([0, legendWidth]);
      legend.append("g")
        .attr("transform", `translate(0, ${legendHeight + 12})`)
        .call(d3.axisBottom(scale).ticks(6)
        .tickFormat(d3.format(".1f")))
        .call(g => g.select(".domain").remove());

      return svg.node();
    }

    // WORK INSHALLAH
    display(resize((width) => stateDelayMap(width)));
  } else {
    // why won't this work????
    display(html`<div style="color:red; padding:20px;">Failed to convert TopoJSON to GeoJSON. Check the map data source.</div>`);
  }
}
```

The map reveals a pattern where states in the Northeast, including Maine, New York, and Massachusetts, show the highest average delays, while states in the Mountain West, such as Idaho, Montana, and Wyoming, show the lowest. The Midwest and mid-Atlantic regions fall in the middle range. This aligned with expectations that the concentration of highly populous cities of the Northeast would increase flight volume, thereby increasing delay volume as well.

The hover tooltip provides specific metrics: the average delay per flight, the total delay minutes accumulated across all flights in that state, and the total number of flights. Illinois, for example, recorded 14.25 minutes of average delay per flight across 9.5 million flights over the period, making it one of the states with the most fligth delays.

This approach indentifies which states experienced the longest delays on average. However, this analysis offers limited insight as this choropleth basically confirms more populated states have more delay volume as a result of them having more flight volume. A state with twice as many flights will accumulate roughly twice the total delay minutes, regardless of whether individual flights are delayed more severely. To account for this relationship, a normalized analysis would be needed to isolate the effect of state-level factors independent of flight volume.

---

# Flight Delays by State per Capita
Normalizing to per capita shifts the pattern. States with large populations and high absolute delay volumes like Illinois, New York, and California now appear lighter on the map, as their delays are distributed across millions of residents. On the other hand, less populous states in the Mountain West and parts of the North like Wyoming and South Dakota show worse performance in terms of flight delays. 

```js
// Load flight data
import * as d3 from "npm:d3";
import {geoAlbersUsa, geoPath} from "npm:d3-geo";
import {resize, html} from "npm:@observablehq/stdlib";
import * as topojson from "npm:topojson";

const data = await FileAttachment("data/flights.csv").csv({typed: true});

// Extract state from airport_name
function extractState(airportName) {
  const match = airportName?.match(/,\s*([A-Z]{2})\s*:/);
  return match ? match[1] : null;
}

// Aggregate total delay minutes AND total flights per state
const stateMap = new Map(); // state -> { totalDelay, totalFlights }
for (const row of data) {
  const state = extractState(row.airport_name);
  if (!state) continue;
  const delay = +row.arr_delay || 0;
  const flights = +row.arr_flights || 0;
  const existing = stateMap.get(state);
  if (existing) {
    existing.totalDelay += delay;
    existing.totalFlights += flights;
  } else {
    stateMap.set(state, { totalDelay: delay, totalFlights: flights });
  }
}

// Population data (millions) – adjust as needed
const statePopulations = {
  "AL":5.1, "AK":0.73, "AZ":7.4, "AR":3.0, "CA":39.0, "CO":5.8, "CT":3.6,
  "DE":1.0, "DC":0.67, "FL":22.6, "GA":10.9, "HI":1.44, "ID":1.96, "IL":12.5,
  "IN":6.8, "IA":3.2, "KS":2.9, "KY":4.5, "LA":4.6, "ME":1.39, "MD":6.2,
  "MA":7.0, "MI":10.0, "MN":5.7, "MS":2.9, "MO":6.2, "MT":1.12, "NE":2.0,
  "NV":3.2, "NH":1.4, "NJ":9.3, "NM":2.1, "NY":19.6, "NC":10.7, "ND":0.78,
  "OH":11.8, "OK":4.0, "OR":4.2, "PA":13.0, "RI":1.1, "SC":5.3, "SD":0.91,
  "TN":7.1, "TX":30.5, "UT":3.4, "VT":0.65, "VA":8.7, "WA":7.8, "WV":1.77,
  "WI":5.9, "WY":0.58
};

// Compute metric: (totalDelay / totalFlights) / population * 1000 (minutes per flight per 1000 people)
const airportTableCap = Array.from(stateMap.entries()).map(([state, { totalDelay, totalFlights }]) => {
  const pop = statePopulations[state] || 1;
  const avgDelayPerFlight = totalFlights > 0 ? totalDelay / totalFlights : 0;
  const delayPerFlightPerCapita = avgDelayPerFlight / pop;   // per person (not scaled)
  return { state, totalDelay, totalFlights, avgDelayPerFlight, delayPerFlightPerCapita };
});

if (airportTableCap.length === 0) {
  display(html`<div style="color:red;">No state data found.</div>`);
} else {
  // Load US TopoJSON and convert to GeoJSON
  let geojson;
  try {
    const topo = await fetch("https://cdn.jsdelivr.net/npm/us-atlas@3/states-10m.json").then(r => r.json());
    geojson = topojson.feature(topo, topo.objects.states);
  } catch(err) {
    display(html`<div style="color:red;">Map data error: ${err.message}</div>`);
    geojson = null;
  }

  if (geojson && geojson.features) {
    function perCapitaMap(width) {
      const height = 720;
      const legendSpace = 110;
      const mapHeight = height - legendSpace;

      const svg = d3.create("svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", `0 0 ${width} ${height}`)
        .style("overflow", "visible");

      const fipsToAbbr = {
        "01":"AL","02":"AK","04":"AZ","05":"AR","06":"CA","08":"CO","09":"CT",
        "10":"DE","11":"DC","12":"FL","13":"GA","15":"HI","16":"ID","17":"IL",
        "18":"IN","19":"IA","20":"KS","21":"KY","22":"LA","23":"ME","24":"MD",
        "25":"MA","26":"MI","27":"MN","28":"MS","29":"MO","30":"MT","31":"NE",
        "32":"NV","33":"NH","34":"NJ","35":"NM","36":"NY","37":"NC","38":"ND",
        "39":"OH","40":"OK","41":"OR","42":"PA","44":"RI","45":"SC","46":"SD",
        "47":"TN","48":"TX","49":"UT","50":"VT","51":"VA","53":"WA","54":"WV",
        "55":"WI","56":"WY"
      };

      const statsByState = new Map();
      for (const row of airportTableCap) {
        statsByState.set(row.state, {
          totalDelay: row.totalDelay,
          totalFlights: row.totalFlights,
          avgDelayPerFlight: row.avgDelayPerFlight,
          delayPerFlightPerCapita: row.delayPerFlightPerCapita
        });
      }

      const values = Array.from(statsByState.values(), d => d.delayPerFlightPerCapita);
      const color = d3.scaleSequential()
        .domain([d3.quantile(values, 0.05), d3.quantile(values, 0.95)])
        .interpolator(t => d3.interpolateRgb("white", "maroon")(t))
        .clamp(true);

      const projection = geoAlbersUsa()
        .scale(1200)
        .translate([width / 2, mapHeight / 2 + 20]);

      const path = geoPath(projection);
      const mapLayer = svg.append("g");

      mapLayer.selectAll("path")
        .data(geojson.features)
        .join("path")
        .attr("d", d => path(d))
        .attr("fill", d => {
          const fips = String(d.id).padStart(2, "0");
          const abbr = fipsToAbbr[fips];
          const s = statsByState.get(abbr);
          return s == null ? "#eee" : color(s.delayPerFlightPerCapita);
        })
        .attr("stroke", "#555")
        .attr("stroke-width", 0.5)
        .on("mousemove", function(event, d) {
          const fips = String(d.id).padStart(2, "0");
          const abbr = fipsToAbbr[fips];
          const s = statsByState.get(abbr);
          const name = d.properties?.name || "Unknown";
          showTooltip(name, abbr, s, event);
        })
        .on("mouseleave", hideTooltip);

      // Tooltip
      const tooltip = svg.append("g")
        .style("pointer-events", "none")
        .attr("opacity", 0);
      const tipRect = tooltip.append("rect")
        .attr("fill", "white")
        .attr("stroke", "#333")
        .attr("rx", 4)
        .attr("ry", 4)
        .attr("opacity", 0.95);
      const tipText = tooltip.append("text")
        .attr("font-size", 14)
        .attr("x", 8)
        .attr("y", 18)
        .attr("font-weight", 600);

      function showTooltip(name, abbr, s, event) {
        const delayMetric = s?.delayPerFlightPerCapita ?? 0;
        const totalDelay = s?.totalDelay ?? 0;
        const avgDelay = s?.avgDelayPerFlight ?? 0;
        const lines = [
          `${name} (${abbr})`,
          `${delayMetric.toFixed(2)} delay minutes per flight per capita`,
          `${avgDelay.toFixed(2)} minutes delay per flight`,
          `${Math.round(totalDelay).toLocaleString()} total delay minutes`
        ];
        tipText.selectAll("tspan").remove();
        lines.forEach((line, i) => {
          tipText.append("tspan")
            .attr("x", 8)
            .attr("dy", i === 0 ? 0 : 16)
            .text(line);
        });
        const bbox = tipText.node().getBBox();
        tipRect
          .attr("width", bbox.width + 16)
          .attr("height", bbox.height + 12);
        const [x, y] = d3.pointer(event, svg.node());
        tooltip
          .attr("transform", `translate(${x + 15},${y - 10})`)
          .attr("opacity", 1);
      }

      function hideTooltip() { tooltip.attr("opacity", 0); }

      // Title
      svg.append("text")
        .attr("x", width / 2)
        .attr("y", 34)
        .attr("text-anchor", "middle")
        .attr("font-size", 24)
        .attr("font-weight", 700)
        .attr("fill", "white")
        .text("Average Delay Minutes per Flight per Capita by State, 2003-2025");

      // Legend
      const legendWidth = 340, legendHeight = 16;
      const legendX = width - legendWidth - 80, legendY = height - 70;
      const legend = svg.append("g")
        .attr("transform", `translate(${legendX}, ${legendY})`);
      const defs = svg.append("defs");
      const gradId = "delayGradRed";
      const grad = defs.append("linearGradient")
        .attr("id", gradId);
      grad.append("stop")
        .attr("offset", "0%")
        .attr("stop-color", color(color.domain()[0]));
      grad.append("stop")
        .attr("offset", "100%")
        .attr("stop-color", color(color.domain()[1]));
      legend.append("text")
        .attr("x", 0)
        .attr("y", -10)
        .attr("font-size", 16)
        .attr("font-weight", 700)
        .attr("fill", "white")
        .text("Delay minutes per flight per capita");
      legend.append("rect")
        .attr("width", legendWidth)
        .attr("height", legendHeight)
        .attr("fill", `url(#${gradId})`)
        .attr("stroke", "#444");
      const scale = d3.scaleLinear()
        .domain(color.domain())
        .range([0, legendWidth]);
      legend.append("g")
        .attr("transform", `translate(0, ${legendHeight + 12})`)
        .call(d3.axisBottom(scale)
          .ticks(6)
          .tickFormat(d3.format(".0f")))
        .call(g => g.select(".domain").remove());

      return svg.node();
    }

    display(resize((width) => perCapitaMap(width)));
  } else {
    display(html`<div style="color:red;">Could not load map geometry.</div>`);
  }
}
```

The normalization reveals the states with the most significant delay burden relative to their populations are actually smaller, less densely populated regions, rather than the major travel hubs that dominated the previous analysis.

<style>

.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: var(--sans-serif);
  margin: 4rem 0 8rem;
  text-wrap: balance;
  text-align: center;
}

.hero h1 {
  margin: 1rem 0;
  padding: 1rem 0;
  max-width: none;
  font-size: 14vw;
  font-weight: 900;
  line-height: 1;
  background: linear-gradient(30deg, var(--theme-foreground-focus), currentColor);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.hero h2 {
  margin: 0;
  max-width: 34em;
  font-size: 20px;
  font-style: initial;
  font-weight: 500;
  line-height: 1.5;
  color: var(--theme-foreground-muted);
}

@media (min-width: 640px) {
  .hero h1 {
    font-size: 90px;
  }
}

</style>
