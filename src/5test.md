---
theme: dashboard
title: Test Flight Delays ⏳
toc: true
---
 
# Flight Delays ⏳


```js
// Load dataset
const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });

```


## Percentage of Delays by Reason 📊

```js
console.log("🚀 Interactive Airport Delay Map Running!");

// ✅ Load necessary D3 libraries
const d3 = await import("https://cdn.jsdelivr.net/npm/d3@7/+esm");
const topojson = await import("https://cdn.jsdelivr.net/npm/topojson@3/+esm");

// ✅ Load US States GeoJSON
const usMap = await d3.json("https://cdn.jsdelivr.net/npm/us-atlas@3/states-10m.json");

// ✅ Assign Seasons to Flights
function getSeason(date) {
  const month = new Date(date).getMonth() + 1;
  if ([12, 1, 2].includes(month)) return "Winter";
  if ([3, 4, 5].includes(month)) return "Spring";
  if ([6, 7, 8].includes(month)) return "Summer";
  return "Fall"; 
}

datasetFlights.forEach(d => {
  d.FL_DATE = new Date(d.FL_DATE);
  d.SEASON = getSeason(d.FL_DATE);
});

// ✅ Compute Average Delay per Airport
const airportDelays = d3.rollups(
  datasetFlights,
  v => ({
    avgDelay: d3.mean(v, d => d.ARR_DELAY),
    totalFlights: v.length
  }),
  d => d.ORIGIN
).map(([airport, stats]) => ({
  airport,
  avgDelay: stats.avgDelay,
  totalFlights: stats.totalFlights
}));

// ✅ Load Airport Coordinates (Replace this with an actual airport dataset)
const airportCoords = {
  "ATL": [-84.4281, 33.6367], "DFW": [-97.0381, 32.8998], "ORD": [-87.9048, 41.9786],
  "DEN": [-104.6737, 39.8561], "LAX": [-118.4085, 33.9416], "JFK": [-73.7781, 40.6413],
  "SFO": [-122.3790, 37.6213], "SEA": [-122.3088, 47.4502], "MIA": [-80.2870, 25.7959],
  "LAS": [-115.1523, 36.0840], "BOS": [-71.0052, 42.3656], "PHX": [-112.0116, 33.4342],
  "IAH": [-95.3414, 29.9844], "MSP": [-93.2218, 44.8810], "DTW": [-83.3534, 42.2124],
  "EWR": [-74.1686, 40.6895], "CLT": [-80.9431, 35.2140], "DCA": [-77.0377, 38.8512],
  "LGA": [-73.8726, 40.7769], "SLC": [-111.9778, 40.7899], "BWI": [-76.6684, 39.1754],
  "TPA": [-82.5332, 27.9755], "PDX": [-122.5975, 45.5898], "STL": [-90.3786, 38.7487],
  "SAN": [-117.1973, 32.7336], "MCO": [-81.3081, 28.4312], "HNL": [-157.9242, 21.3187],
  "DAL": [-96.8518, 32.8471], "MDW": [-87.7524, 41.7868], "FLL": [-80.1449, 26.0726],
  "AUS": [-97.6699, 30.1975], "RDU": [-78.7875, 35.8801], "IND": [-86.2944, 39.7173],
  "BNA": [-86.6782, 36.1263], "CMH": [-82.8813, 39.9979], "PIT": [-80.2329, 40.4915],
  "MSY": [-90.2580, 29.9934], "SMF": [-121.5908, 38.6951], "SAT": [-98.4727, 29.5337],
  "SJC": [-121.9290, 37.3626], "CLE": [-81.8498, 41.4101], "HOU": [-95.2789, 29.6454],
  "JAX": [-81.6879, 30.4940], "OMA": [-95.8941, 41.3032], "OKC": [-97.6007, 35.3931],
  "MEM": [-89.9767, 35.0424], "SNA": [-117.8674, 33.6757], "ONT": [-117.6012, 34.0560],
  "BUR": [-118.3527, 34.2007], "RSW": [-81.7552, 26.5362], "TUL": [-95.8881, 36.1984],
  "BOI": [-116.2229, 43.5644], "ELP": [-106.3778, 31.8072], "RIC": [-77.3232, 37.5052],
  "MHT": [-71.4382, 42.9326], "LIT": [-92.2243, 34.7294], "PBI": [-80.0956, 26.6832],
  "SAV": [-81.2021, 32.1276], "GSP": [-82.2189, 34.8956], "ALB": [-73.8017, 42.7483],
  "CHS": [-80.0405, 32.8986], "TUS": [-110.9410, 32.1161], "GRR": [-85.5228, 42.8809],
  "PSP": [-116.5085, 33.8297], "PWM": [-70.3093, 43.6462], "MSN": [-89.3375, 43.1399],
  "COS": [-104.7003, 38.8058], "FAT": [-119.7180, 36.7762], "DAY": [-84.2194, 39.9024],
  "ICT": [-97.4309, 37.6499], "SDF": [-85.7365, 38.1744], "XNA": [-94.3068, 36.2819],
  "GEG": [-117.5338, 47.6281], "BTV": [-73.1533, 44.4694], "ABE": [-75.4408, 40.6521],
  "MOB": [-88.2428, 30.6914], "SRQ": [-82.5530, 27.3954], "TLH": [-84.3504, 30.3965],
  "TYS": [-83.9933, 35.8128], "AVL": [-82.5418, 35.4362], "SYR": [-76.1071, 43.1112],
  "BIL": [-108.5429, 45.8077], "CAK": [-81.4422, 40.9161], "LBB": [-101.8234, 33.6636],
  "GPT": [-89.0720, 30.4120], "ECP": [-85.7956, 30.3571], "BZN": [-111.1524, 45.7770]
};


// ✅ Merge Delay Data with Coordinates
const airportData = airportDelays
  .filter(d => airportCoords[d.airport])
  .map(d => ({
    airport: d.airport,
    avgDelay: d.avgDelay,
    totalFlights: d.totalFlights,
    coords: airportCoords[d.airport]
  }));

console.log("🛠 Processed Airport Data:", airportData);

// ✅ Set Map Dimensions
const width = 950, height = 600;

// ✅ Projection & Path Generator
const projection = d3.geoAlbersUsa().fitSize([width, height], topojson.feature(usMap, usMap.objects.states));
const path = d3.geoPath().projection(projection);

// ✅ Select Container & Remove Old SVG
const container = d3.select("#airport-map-container");
container.select("svg").remove();

// ✅ Create SVG
const svg = container.append("svg")
  .attr("width", width)
  .attr("height", height)
  .style("font", "12px sans-serif");

// ✅ Tooltip
const tooltip = d3.select("body").append("div")
  .attr("class", "tooltip")
  .style("position", "absolute")
  .style("background", "rgba(0, 0, 0, 0.9)")
  .style("color", "white")
  .style("padding", "6px 10px")
  .style("border-radius", "5px")
  .style("font-size", "12px")
  .style("pointer-events", "none")
  .style("display", "none");

// ✅ Draw US States
svg.append("g")
  .selectAll("path")
  .data(topojson.feature(usMap, usMap.objects.states).features)
  .join("path")
  .attr("d", path)
  .attr("fill", "#2c3e50") // Dark background for map
  .attr("stroke", "#fff");

// ✅ Define Color Scale for Delays
const colorScale = d3.scaleDiverging()
  .domain([-10, 0, 30]) // Negative = Early, 0 = On Time, 30+ = Very Late
  .interpolator(d3.interpolateRdYlGn); // Red = Late, White = On-time, Green = Early

// ✅ Define Size Scale for Flights
const sizeScale = d3.scaleSqrt()
  .domain([0, d3.max(airportData, d => d.totalFlights)])
  .range([5, 30]); // Circle size

// ✅ Draw Airport Circles
svg.append("g")
  .selectAll("circle")
  .data(airportData)
  .join("circle")
  .attr("cx", d => projection(d.coords)[0])
  .attr("cy", d => projection(d.coords)[1])
  .attr("r", d => sizeScale(d.totalFlights))
  .attr("fill", d => colorScale(d.avgDelay))
  .attr("stroke", "#222")
  .attr("opacity", 0.8)
  .on("mouseover", function (event, d) {
    d3.select(this).attr("stroke", "white");
    tooltip.style("display", "block")
      .html(`
        <strong>${d.airport}</strong><br>
        ⏳ Avg Delay: ${Math.round(d.avgDelay)} mins<br>
        ✈ Flights: ${d.totalFlights}
      `);
  })
  .on("mousemove", event => {
    tooltip.style("top", `${event.pageY + 10}px`).style("left", `${event.pageX + 10}px`);
  })
  .on("mouseout", function () {
    d3.select(this).attr("stroke", "#222");
    tooltip.style("display", "none");
  });

// ✅ Create Season Toggle
const selectedSeason = Inputs.radio(["All", "Winter", "Spring", "Summer", "Fall"], {
  label: "🌍 Select Season",
  value: "All"
});

// ✅ Function to Update Map Based on Season
function updateMap() {
  const filteredData = selectedSeason.value === "All"
    ? airportData
    : airportData.filter(d => datasetFlights.some(f => f.ORIGIN === d.airport && f.SEASON === selectedSeason.value));

  svg.selectAll("circle")
    .data(filteredData, d => d.airport)
    .join(
      enter => enter.append("circle")
        .attr("cx", d => projection(d.coords)[0])
        .attr("cy", d => projection(d.coords)[1])
        .attr("r", d => sizeScale(d.totalFlights))
        .attr("fill", d => colorScale(d.avgDelay))
        .attr("stroke", "#222")
        .attr("opacity", 0.8)
        .on("mouseover", function (event, d) {
          d3.select(this).attr("stroke", "white");
          tooltip.style("display", "block")
            .html(`
              <strong>${d.airport}</strong><br>
              ⏳ Avg Delay: ${Math.round(d.avgDelay)} mins<br>
              ✈ Flights: ${d.totalFlights}
            `);
        })
        .on("mousemove", event => {
          tooltip.style("top", `${event.pageY + 10}px`).style("left", `${event.pageX + 10}px`);
        })
        .on("mouseout", function () {
          d3.select(this).attr("stroke", "#222");
          tooltip.style("display", "none");
        }),
      update => update.transition().duration(500)
        .attr("r", d => sizeScale(d.totalFlights))
        .attr("fill", d => colorScale(d.avgDelay))
    );
}

// ✅ Listen for Season Toggle Changes
selectedSeason.addEventListener("input", updateMap);

// ✅ Initial Map Render
updateMap();



```


<div class="grid grid-cols-1">
  <div class="card">
    <div id="airport-map-container"></div> <!-- Bar Chart Container -->
  </div>
</div>

<p>

Airports with negative delays (flights arrive early) → Green

Airports with 0 delay (on time) → White

Airports with high average delays (30+ mins) → Red


This ensures that:

Red indicates airports with severe delays.

Green indicates airports where flights tend to arrive early.

White or Yellowish means the airport is closer to 0 average delay (on-time flights).

If an airport is a different shade from others, it means its average delay is significantly different from the others.

</p>