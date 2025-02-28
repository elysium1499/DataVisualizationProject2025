---
theme: dashboard
title: Test
toc: true
---


<br>


## Flight Distance vs. Average Delay per Airline

```js
console.log("🚀 Scatter plot function is running!"); 

// Load dataset
const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });
console.log("🛠 Data loaded:", datasetFlights); // Debugging

// ✅ Load necessary D3 libraries
const d3 = await import("https://cdn.jsdelivr.net/npm/d3@7/+esm");

// ✅ Make a deep copy of dataset to avoid mutations
const scatterData = JSON.parse(JSON.stringify(datasetFlights));

// ✅ Process Data for Scatter Plot
const processedScatterData = scatterData.map(d => ({
  airline: d.AIRLINE,
  distance: +d.DISTANCE, // Convert to number
  delay: +d.ARR_DELAY    // Convert to number
})).filter(d => !isNaN(d.distance) && !isNaN(d.delay)); // ✅ Remove NaN values

console.log("🛠 Processed Data Sample:", processedScatterData.slice(0, 5)); // ✅ Debugging

// ✅ Compute average delay & distance per airline
const scatterStats = d3.rollups(
  processedScatterData,
  v => ({
    avgDistance: d3.mean(v, d => d.distance),
    avgDelay: d3.mean(v, d => d.delay),
    numFlights: v.length
  }),
  d => d.airline
).map(([airline, stats]) => ({
  airline,
  ...stats
}));

console.log("📊 Processed Scatter Stats:", scatterStats); // ✅ Debugging

// ✅ Step 1: Set up chart dimensions
const scatterWidth = 900, scatterHeight = 600;
const scatterMargin = { top: 50, right: 50, bottom: 80, left: 80 };

// ✅ Step 2: Define Scales
const scatterXScale = d3.scaleLinear()
  .domain([0, d3.max(scatterStats, d => d.avgDistance) || 1000]) // ✅ Default value to avoid undefined
  .range([scatterMargin.left, scatterWidth - scatterMargin.right]);

const scatterYScale = d3.scaleLinear()
  .domain([d3.min(scatterStats, d => d.avgDelay) - 1, d3.max(scatterStats, d => d.avgDelay) || 100]) // ✅ Default value to avoid undefined
  .range([scatterHeight - scatterMargin.bottom, scatterMargin.top]);

const scatterSizeScale = d3.scaleSqrt()
  .domain([0, d3.max(scatterStats, d => d.numFlights) || 1000]) // ✅ Default value to avoid undefined
  .range([5, 20]); // Point size based on number of flights

const scatterColorScale = d3.scaleOrdinal(d3.schemeTableau10)
  .domain(scatterStats.map(d => d.airline));

// ✅ Step 3: Select the container div
const scatterContainer = d3.select("#scatter-container");

// ✅ Remove old SVG to prevent duplication
scatterContainer.select("svg").remove();

// ✅ Step 4: Create SVG
const scatterSvg = scatterContainer.append("svg")
  .attr("width", scatterWidth)
  .attr("height", scatterHeight)
  .style("font", "12px sans-serif");

// ✅ Tooltip
const scatterTooltip = d3.select("body").append("div")
  .attr("class", "tooltip")
  .style("position", "absolute")
  .style("background", "rgba(0, 0, 0, 0.8)")
  .style("color", "white")
  .style("padding", "6px 10px")
  .style("border-radius", "5px")
  .style("font-size", "12px")
  .style("pointer-events", "none")
  .style("display", "none");

// ✅ Step 5: Draw Scatter Plot
scatterSvg.append("g")
  .selectAll("circle")
  .data(scatterStats)
  .join("circle")
  .attr("cx", d => scatterXScale(d.avgDistance))
  .attr("cy", d => scatterYScale(d.avgDelay))
  .attr("r", d => scatterSizeScale(d.numFlights))
  .attr("fill", d => scatterColorScale(d.airline))
  .attr("opacity", 0.9)
  .on("mouseover", function (event, d) {
    d3.select(this).attr("stroke", "white").attr("stroke-width", 2);
    scatterTooltip.style("display", "block")
      .html(`
        <strong>${d.airline}</strong><br>
        📏 Avg Distance: ${Math.round(d.avgDistance)} miles<br>
        ⏳ Avg Delay: ${Math.round(d.avgDelay)} mins<br>
        ✈ Flights: ${d.numFlights}
      `);
  })
  .on("mousemove", event => {
    scatterTooltip.style("top", `${event.pageY + 10}px`).style("left", `${event.pageX + 10}px`);
  })
  .on("mouseout", function () {
    d3.select(this).attr("stroke", "none");
    scatterTooltip.style("display", "none");
  });

// ✅ Step 6: Add Axes
scatterSvg.append("g")
  .attr("transform", `translate(0,${scatterHeight - scatterMargin.bottom})`)
  .call(d3.axisBottom(scatterXScale))
  .append("text")
  .attr("x", scatterWidth / 2)
  .attr("y", 40)
  .attr("fill", "white")
  .attr("text-anchor", "middle")
  .style("font-size", "14px")
  .text("Average Flight Distance (miles)");

scatterSvg.append("g")
  .attr("transform", `translate(${scatterMargin.left},0)`)
  .call(d3.axisLeft(scatterYScale))
  .append("text")
  .attr("x", -scatterHeight / 2)
  .attr("y", -50)
  .attr("transform", "rotate(-90)")
  .attr("fill", "white")
  .attr("text-anchor", "middle")
  .style("font-size", "14px")
  .text("Average Delay (minutes)");

// ✅ Add a dotted reference line at y = 0
scatterSvg.append("line")
  .attr("x1", scatterMargin.left)
  .attr("x2", scatterWidth - scatterMargin.right)
  .attr("y1", scatterYScale(0)) // Map 0 to the Y scale
  .attr("y2", scatterYScale(0))
  .attr("stroke", "white") // Line color (Change to preferred color)
  .attr("stroke-width", 1)
  .attr("stroke-dasharray", "5,5"); // Dotted line pattern



// ✅ Step 7: Add Title
//scatterSvg.append("text")
//  .attr("x", scatterWidth / 2)
//  .attr("y", 20)
//  .attr("text-anchor", "middle")
//  .style("font-size", "16px")
//  .style("fill", "white")
  //.text("Flight Distance vs. Average Delay per Airline");

```

<div class="grid grid-cols-1">
  <div class="card">
    <div id="scatter-container"></div> <!-- Scatter plot container  -->
  </div>
</div> 

<!-- <div class="grid grid-cols-1">
  <div class="card">
    <div id="scatter-container"></div> <!-- Scatter plot container 
  </div>
</div> > -->

<br>

# Average Delay per Airline (Bubble Size = Flights)
```js
console.log("🚀 Scatter plot function is running!");


// ✅ Ensure dataset is properly processed
const scatterDataset = datasetFlights.map(d => ({
  airline: d.AIRLINE,
  delay: +d.ARR_DELAY, // Convert to number
  numFlights: 1  // Count each flight for later aggregation
})).filter(d => !isNaN(d.delay)); // ✅ Remove NaN values

console.log("🛠 Processed Data Sample:", scatterDataset.slice(0, 5)); // ✅ Debugging

// ✅ Compute average delay per airline
const airlineStats = d3.rollups(
  scatterDataset,
  v => ({
    avgDelay: d3.mean(v, d => d.delay), // Compute average delay
    numFlights: v.length // Count number of flights per airline
  }),
  d => d.airline
).map(([airline, stats]) => ({
  airline,
  ...stats
}));

console.log("📊 Processed Airline Stats:", airlineStats); // ✅ Debugging

// ✅ Step 1: Set up chart dimensions
const width = 900, height = 600;
const margin = { top: 50, right: 50, bottom: 180, left: 80 }; // Increased bottom margin for airline names

// ✅ Step 2: Define Scales
const xScale = d3.scaleBand()
  .domain(airlineStats.map(d => d.airline))
  .range([margin.left, width - margin.right])
  .padding(0.4); // ✅ Categorical x-axis with padding

const yScale = d3.scaleLinear()
  .domain([-4, d3.max(airlineStats, d => d.avgDelay) || 100]) // ✅ Default value to avoid undefined
  .nice()
  .range([height - margin.bottom, margin.top]);

const sizeScale = d3.scaleSqrt()
  .domain([0, d3.max(airlineStats, d => d.numFlights) || 1000]) // ✅ Default value to avoid undefined
  .range([5, 20]); // Point size based on number of flights

const colorScale = d3.scaleOrdinal(d3.schemeTableau10)
  .domain(airlineStats.map(d => d.airline));

// ✅ Step 3: Select the container div
const container = d3.select("#scatter-container2");

// ✅ Remove old SVG to prevent duplication
container.select("svg").remove();

// ✅ Step 4: Create SVG
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

// ✅ Step 5: Draw Scatter Plot
svg.append("g")
  .selectAll("circle")
  .data(airlineStats)
  .join("circle")
  .attr("cx", d => xScale(d.airline) + xScale.bandwidth() / 2) // Center bubbles
  .attr("cy", d => yScale(d.avgDelay))
  .attr("r", d => sizeScale(d.numFlights))
  .attr("fill", d => colorScale(d.airline))
  .attr("opacity", 0.9)
  .on("mouseover", function (event, d) {
    d3.select(this).attr("stroke", "white").attr("stroke-width", 2);
    tooltip.style("display", "block")
      .html(`
        <strong>${d.airline}</strong><br>
        ⏳ Avg Delay: ${Math.round(d.avgDelay)} mins<br>
        ✈ Flights: ${d.numFlights}
      `);
  })
  .on("mousemove", event => {
    tooltip.style("top", `${event.pageY + 10}px`).style("left", `${event.pageX + 10}px`);
  })
  .on("mouseout", function () {
    d3.select(this).attr("stroke", "none");
    tooltip.style("display", "none");
  });

// ✅ Step 6: Add Axes
svg.append("g")
  .attr("transform", `translate(0,${height - margin.bottom})`)
  .call(d3.axisBottom(xScale).tickSize(0))
  .selectAll("text")
  .attr("transform", "rotate(-45)") // ✅ Rotate x-axis labels to fit better
  .style("text-anchor", "end")
  .style("fill", "white")
  .style("font-size", "10px");

svg.append("g")
  .attr("transform", `translate(${margin.left},0)`)
  .call(d3.axisLeft(yScale))
  .selectAll("text")
  .style("fill", "white")
  .style("font-size", "12px");

// ✅ Step 7: Add Labels
svg.append("text")
  .attr("x", width / 2)
  .attr("y", height -70)
  .attr("fill", "white")
  .attr("text-anchor", "middle")
  .style("font-size", "14px")
  .text("Airline");

svg.append("text")
  .attr("x", -height / 2)
  .attr("y", 30)
  .attr("transform", "rotate(-90)")
  .attr("fill", "white")
  .attr("text-anchor", "middle")
  .style("font-size", "14px")
  .text("Average Delay (minutes)");

// ✅ Add a dotted reference line at y = 0
svg.append("line")
  .attr("x1", margin.left)
  .attr("x2", width - margin.right)
  .attr("y1", yScale(0)) // Map 0 to the Y scale
  .attr("y2", yScale(0))
  .attr("stroke", "white") // Line color (Change to preferred color)
  .attr("stroke-width", 1)
  .attr("stroke-dasharray", "5,5"); // Dotted line pattern


// ✅ Step 8: Add Title
//svg.append("text")
//  .attr("x", width / 2)
//  .attr("y", 20)
//  .attr("text-anchor", "middle")
//  .style("font-size", "16px")
//  .style("fill", "white")
  //.text("Average Delay per Airline (Bubble Size = Flights)");

```
<div class="grid grid-cols-1">
  <div class="card">
    <div id="scatter-container2"></div> <!-- Scatter plot container  -->
  </div>
</div> 
