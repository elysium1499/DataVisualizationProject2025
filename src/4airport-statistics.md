---
theme: dashboard
title: Airport statistics 📍
toc: true
---
 
# Airport statistics 📍

<br>


## Flight Delay Distribution by Airport

### Flight Delay Distribution of Top 20 Busiest Airports (Grouped by State)

```js
// Load dataset
const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });
console.log("🛠 Data loaded:", datasetFlights); // Debugging

console.log("🚀 Box plot function is running!");

// ✅ Load necessary D3 libraries
const d3 = await import("https://cdn.jsdelivr.net/npm/d3@7/+esm");

// ✅ Define IATA Airport Code → State Mapping
//const airportStateMap = {
//  "ATL": "Georgia (GA)", "DFW": "Texas (TX)", "ORD": "Illinois (IL)", "DEN": "Colorado (CO)",
//  "CLT": "North Carolina (NC)", "LAX": "California (CA)", "LAS": "Nevada (NV)", "PHX": "Arizona (AZ)",
//  "SEA": "Washington (WA)", "MCO": "Florida (FL)", "IAH": "Texas (TX)", "DTW": "Michigan (MI)",
//  "LGA": "New York (NY)", "MSP": "Minnesota (MN)", "SFO": "California (CA)", "BOS": "Massachusetts (MA)",
//  "EWR": "New Jersey (NJ)", "DCA": "Virginia (VA)", "JFK": "New York (NY)", "SLC": "Utah (UT)"
//};

const airportStateMap = {
  "ATL": "Georgia", "DFW": "Texas", "ORD": "Illinois", "DEN": "Colorado",
  "CLT": "North Carolina", "LAX": "California", "LAS": "Nevada", "PHX": "Arizona",
  "SEA": "Washington", "MCO": "Florida", "IAH": "Texas", "DTW": "Michigan",
  "LGA": "New York", "MSP": "Minnesota", "SFO": "California", "BOS": "Massachusetts",
  "EWR": "New Jersey", "DCA": "Virginia", "JFK": "New York", "SLC": "Utah"
};

// ✅ Count Flights Per Airport
const airportFlightCounts = d3.rollups(
  datasetFlights,
  v => v.length,
  d => d.ORIGIN
);

// ✅ Sort Airports & Select the 20 Busiest Airports
const topAirports = new Set(
  airportFlightCounts.sort((a, b) => b[1] - a[1]) // Sort descending
    .slice(0, 20) // Take top 20
    .map(d => d[0]) // Extract airport codes
);

// ✅ Filter Dataset to Keep Only the 20 Busiest Airports
const filteredDataset = datasetFlights.filter(d => topAirports.has(d.ORIGIN));

// ✅ Process Data: Compute Summary Statistics for Each Airport
const airportStats = d3.rollups(
  filteredDataset,
  v => ({
    minDelay: d3.min(v, d => d.ARR_DELAY),
    q1: d3.quantile(v.map(d => d.ARR_DELAY).sort(d3.ascending), 0.25),
    median: d3.median(v, d => d.ARR_DELAY),
    q3: d3.quantile(v.map(d => d.ARR_DELAY).sort(d3.ascending), 0.75),
    maxDelay: d3.max(v, d => d.ARR_DELAY)
  }),
  d => d.ORIGIN
).map(([airport, stats]) => ({
  airport,
  state: airportStateMap[airport] || "Unknown", // ✅ Assign State
  ...stats
}));

// ✅ Sort Airports by State First, Then by Airport Name
const sortedAirportStats = airportStats.sort((a, b) => 
  d3.ascending(a.state, b.state) || d3.ascending(a.airport, b.airport)
);

// ✅ Set up Dimensions
const width = 1000, height = 650, margin = { top: 50, right: 50, bottom: 150, left: 80 };

// ✅ Define Scales
const xScale = d3.scaleBand()
  .domain(sortedAirportStats.map(d => d.airport)) // Sorted Airports
  .range([margin.left, width - margin.right])
  .padding(0.3);

const yScale = d3.scaleLinear()
  .domain([-70, 60]) // ✅ Standardized Delay Range
  .nice()
  .range([height - margin.bottom, margin.top]);

// ✅ Assign Colors to States
//const colorScale = d3.scaleOrdinal(d3.schemeCategory10)
//  .domain([...new Set(sortedAirportStats.map(d => d.state))]);
// ✅ Define Color Scale for States with 17 Unique Colors
const baseColors = d3.schemeCategory10; // 12 Vibrant Colors from D3
//const baseColors = d3.schemeSet3; // 12 Vibrant Colors from D3
const extraColors = d3.range(8).map(i => d3.interpolateRainbow(i / 8)); // Generate 5 more

const fullColorPalette = [...baseColors, ...extraColors]; // Merge Colors
const colorScale = d3.scaleOrdinal(fullColorPalette)
  .domain([...new Set(sortedAirportStats.map(d => d.state))]); // Assign to states



// ✅ Select Container
const container = d3.select("#boxplot-container");

// ✅ Remove Old SVG
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
  .style("background", "rgba(0, 0, 0, 0.8)")
  .style("color", "white")
  .style("padding", "6px 10px")
  .style("border-radius", "5px")
  .style("font-size", "12px")
  .style("pointer-events", "none")
  .style("display", "none");

// ✅ Draw Box Plot for Each Airport
svg.append("g")
  .selectAll("g")
  .data(sortedAirportStats)
  .join("g")
  .each(function(d) {
    const g = d3.select(this);
    //const xPos = xScale(d.airport);
    const xPos = xScale(d.airport) + xScale.bandwidth() / 2; // Center whiskers
    const color = colorScale(d.state);
    const boxWidth = xScale.bandwidth();

    // ✅ Draw Box
    g.append("rect")
      .attr("x", xPos - boxWidth / 2)
      .attr("y", yScale(d.q3))
      .attr("height", yScale(d.q1) - yScale(d.q3))
      .attr("width", xScale.bandwidth())
      .attr("fill", color)
      .attr("opacity", 0.6)
      .on("mouseover", function(event) {
        d3.select(this).style("opacity", 1);
        tooltip.style("display", "block")
          .html(`
            <strong>${d.airport} - ${d.state}</strong><br>
            Min: ${d.minDelay} mins<br>
            Q1: ${Math.round(d.q1)} mins<br>
            Median: ${Math.round(d.median)} mins<br>
            Q3: ${Math.round(d.q3)} mins<br>
            Max: ${d.maxDelay} mins
          `);
      })
      .on("mousemove", event => {
        tooltip.style("top", `${event.pageY + 10}px`).style("left", `${event.pageX + 10}px`);
      })
      .on("mouseout", function() {
        d3.select(this).style("opacity", 0.6);
        tooltip.style("display", "none");
      });

    // ✅ Draw Median Line
    g.append("line")
      .attr("x1", xPos - boxWidth / 2)
      .attr("x2", xPos + boxWidth / 2)
      .attr("y1", yScale(d.median))
      .attr("y2", yScale(d.median))
      .attr("stroke", "white")
      .attr("stroke-width", 2);

    
    // ✅ Compute IQR and Adjust Whisker Bounds
    const iqr = d.q3 - d.q1;
    const lowerWhisker = Math.max(d.q1 - 1.5 * iqr, d.minDelay); // 1.5×IQR Rule
    const upperWhisker = Math.min(d.q3 + 1.5 * iqr, d.maxDelay); // 1.5×IQR Rule

    // ✅ Draw Whiskers (1.5×IQR instead of full min/max)
    g.append("line") // Lower whisker (Restricted Min to Q1)
      .attr("x1", xPos)
      .attr("x2", xPos)
      .attr("y1", yScale(lowerWhisker))
      .attr("y2", yScale(d.q1))
      .attr("stroke", "white")
      .attr("stroke-width", 2);

    g.append("line") // Upper whisker (Q3 to Restricted Max)
      .attr("x1", xPos)
      .attr("x2", xPos)
      .attr("y1", yScale(d.q3))
      .attr("y2", yScale(upperWhisker))
      .attr("stroke", "white")
      .attr("stroke-width", 2);

    // ✅ Add horizontal caps to whiskers
    g.append("line") // Cap at Lower Whisker
      .attr("x1", xPos - 5)
      .attr("x2", xPos + 5)
      .attr("y1", yScale(lowerWhisker))
      .attr("y2", yScale(lowerWhisker))
      .attr("stroke", "white")
      .attr("stroke-width", 2);

    g.append("line") // Cap at Upper Whisker
      .attr("x1", xPos - 5)
      .attr("x2", xPos + 5)
      .attr("y1", yScale(upperWhisker))
      .attr("y2", yScale(upperWhisker))
      .attr("stroke", "white")
      .attr("stroke-width", 2);

    // ✅ Draw Outlier Dots (Points outside 1.5×IQR range)
    g.selectAll(".outlier")
      .data(d.ARR_DELAY > upperWhisker || d.ARR_DELAY < lowerWhisker ? [d.ARR_DELAY] : []) // Only plot outliers
      .join("circle")
      .attr("cx", xPos)
      .attr("cy", d => yScale(d))
      .attr("r", 3)
      .attr("fill", "red");

  });

// ✅ Y Axis (Delays)
svg.append("g")
  .attr("transform", `translate(${margin.left},0)`)
  .call(d3.axisLeft(yScale));

// ✅ X-Axis with Airport Labels
svg.append("g")
  .attr("transform", `translate(0,${height - margin.bottom})`)
  .call(d3.axisBottom(xScale))
  .selectAll("text")
  .attr("transform", "rotate(-45)")
  .style("text-anchor", "end")
  .style("fill", "white");

// ✅ Add **State Labels as Group Titles** Above Airports
const stateGroups = d3.groups(sortedAirportStats, d => d.state);
stateGroups.forEach(([state, airports], i) => {
  const firstAirport = airports[0].airport;
  const lastAirport = airports[airports.length - 1].airport;
  const startX = xScale(firstAirport) + xScale.bandwidth() / 2;
  const endX = xScale(lastAirport) + xScale.bandwidth() / 2;

  // Draw a line to separate state groups
  svg.append("line")
    .attr("x1", startX)
    .attr("x2", endX)
    .attr("y1", height - margin.bottom + 2)
    .attr("y2", height - margin.bottom + 2)
    .attr("stroke", "white")
    .attr("stroke-width", 2);

  // Place the state label above the line
  svg.append("text")
    .attr("x", (startX + endX) / 2)
    .attr("y", height - margin.bottom + 50)
    .attr("fill", "white")
    .attr("text-anchor", "middle")
    .style("font-size", "7.5px")
    //.style("font-weight", "bold")
    //.attr("transform", "rotate(-45)")
    .text(state);
});

```
<div class="grid grid-cols-1">
  <div class="card">
    <div id="boxplot-container"></div> 
  </div>
</div> 

<br>

## Airport Performance on Delays, Cancellations, and Diversions

```js
console.log("🚀 Small Multiples Radar Chart with Airport Labels!");

// ✅ Load necessary D3 libraries
const d3 = await import("https://cdn.jsdelivr.net/npm/d3@7/+esm");

// ✅ Define IATA Airport Code → State Mapping
const airportStateMap = {
  "ATL": "Georgia", "ORD": "Illinois", "LAX": "California", "DFW": "Texas",
  "JFK": "New York", "DEN": "Colorado", "SFO": "California",
  "MCO": "Florida", "SEA": "Washington"
};

// ✅ Manually Selected 9 Airports
const selectedAirports = new Set(["ATL", "ORD", "LAX", "DFW", "JFK", "DEN", "SFO", "MCO", "SEA"]);

// ✅ Filter Dataset to Keep Only the Selected Airports
const filteredDataset = datasetFlights.filter(d => selectedAirports.has(d.ORIGIN));

// ✅ Process Data: Compute Metrics for Each Airport
const airportStats = d3.rollups(
  filteredDataset,
  v => ({
    avgDelay: d3.mean(v, d => d.ARR_DELAY),
    cancellations: d3.mean(v, d => d.CANCELLED) * 100, // Convert to percentage
    diverted: d3.mean(v, d => d.DIVERTED) * 100 // Convert to percentage
  }),
  d => d.ORIGIN
).map(([airport, stats]) => ({
  airport,
  state: airportStateMap[airport] || "Unknown",
  ...stats
}));

// ✅ Get Max Values for Each Metric (New Scaling Approach)
const maxDelay = d3.max(airportStats, d => d.avgDelay);
const maxCancellations = d3.max(airportStats, d => d.cancellations);
const maxDiverted = d3.max(airportStats, d => d.diverted);

// ✅ Set Radar Chart Layout
const numAxes = 3; // Three metrics: Delay, Cancellations, Diversions
const radius = 80; // Size of each radar chart
const angleSlice = (Math.PI * 2) / numAxes; // Divide circle into equal parts

// ✅ Define Colors for Different Metrics
const colorScale = d3.scaleOrdinal(d3.schemeSet2).domain(["avgDelay", "cancellations", "diverted"]);

// ✅ Select the Container
const container = d3.select("#radar-chart-container");

// ✅ Remove Old Charts
container.html("");

// ✅ Create Grid Layout for Small Multiples
const chartWidth = 250, chartHeight = 250;

// ✅ Create SVG for Each Airport
airportStats.forEach((airportData, i) => {
  // ✅ Create a div for each radar chart (for better control)
  const airportDiv = container.append("div")
    .style("display", "inline-block")
    .style("text-align", "center") // ✅ Center labels
    .style("margin", "10px");

  // ✅ Add Airport Label
  airportDiv.append("div")
    .style("color", "white")
    .style("font-size", "12px")
    .style("font-weight", "bold")
    .style("margin-bottom", "5px") // ✅ Space between text and chart
    .text(airportData.airport);

  // ✅ Create SVG
  const svg = airportDiv.append("svg")
    .attr("width", chartWidth)
    .attr("height", chartHeight)
    .style("font", "10px sans-serif")
    .style("margin", "5px");

  const g = svg.append("g")
    .attr("transform", `translate(${chartWidth / 2}, ${chartHeight / 2})`);

  // ✅ Draw Radar Grid (Circular Grid)
  for (let level = 1; level <= 5; level++) {
    g.append("circle")
      .attr("r", (radius / 5) * level)
      .attr("stroke", "#ddd")
      .attr("fill", "none");

    // ✅ Label for each level
    g.append("text")
      .attr("x", 0)
      .attr("y", -(radius / 5) * level)
      .attr("dy", "-4px")
      .attr("text-anchor", "middle")
      .style("fill", "white")
      .style("font-size", "8px")
      .text(Math.round((level / 5) * 100) + "%");
  }

  // ✅ Draw Radar Axes
  const metrics = ["avgDelay", "cancellations", "diverted"];
  const maxValues = { avgDelay: maxDelay, cancellations: maxCancellations, diverted: maxDiverted };

  metrics.forEach((metric, index) => {
    const angle = angleSlice * index;
    const x = Math.cos(angle) * radius;
    const y = Math.sin(angle) * radius;

    g.append("line")
      .attr("x1", 0)
      .attr("y1", 0)
      .attr("x2", x)
      .attr("y2", y)
      .attr("stroke", "#bbb");

    // ✅ Axis Labels
    g.append("text")
      .attr("x", x * 1.2)
      .attr("y", y * 1.2)
      .attr("text-anchor", "middle")
      .attr("dy", "0.35em")
      .style("fill", "white")
      .text(metric === "avgDelay" ? "Avg Delay" : metric === "cancellations" ? "Cancellations" : "Diverted");
  });

  // ✅ Draw Radar Chart Shape (Using **MAX VALUES FOR EACH METRIC**)
  const radarPoints = metrics.map((metric, index) => {
    const value = (airportData[metric] / maxValues[metric]) * radius; // ✅ Normalize based on the max value per metric
    const angle = angleSlice * index;
    return [Math.cos(angle) * value, Math.sin(angle) * value];
  });

  g.append("polygon")
    .attr("points", radarPoints.map(d => d.join(",")).join(" "))
    .attr("fill", colorScale(airportData.airport))
    .attr("opacity", 0.6)
    .attr("stroke", "white")
    .on("mouseover", function (event) {
      d3.select(this).attr("opacity", 1);
      tooltip.style("display", "block")
        .html(`
          <strong>${airportData.airport} (${airportData.state})</strong><br>
          ⏳ Avg Delay: ${Math.round(airportData.avgDelay)} mins<br>
          ❌ Cancellations: ${Math.round(airportData.cancellations)}%<br>
          🔄 Diverted: ${Math.round(airportData.diverted)}%
        `);
    })
    .on("mousemove", event => {
      tooltip.style("top", `${event.pageY + 10}px`).style("left", `${event.pageX + 10}px`);
    })
    .on("mouseout", function () {
      d3.select(this).attr("opacity", 0.6);
      tooltip.style("display", "none");
    });
});


```
<div class="grid grid-cols-1">
  <div class="card">
    <div id="radar-chart-container"></div> 
  </div>
</div> 

<p>

Selected 8 Airports
IATA Code	Airport Name	State	Reason for Selection

ATL	Hartsfield-Jackson Atlanta Intl	Georgia (GA)	Busiest airport in the USA, major Delta hub

ORD	Chicago O’Hare Intl	Illinois (IL)	High delays, major United & American hub

LAX	Los Angeles Intl	California (CA)	One of the busiest West Coast hubs

DFW	Dallas/Fort Worth Intl	Texas (TX)	Largest American Airlines hub

JFK	John F. Kennedy Intl	New York (NY)	Major international gateway

DEN	Denver Intl	Colorado (CO)	Large hub with unique weather challenges

SFO	San Francisco Intl	California (CA)	High delays due to fog/weather

MCO	Orlando Intl	Florida (FL)	High leisure travel volume

SEA	Seattle-Tacoma Intl	Washington (WA)	Important West Coast gateway

</p>