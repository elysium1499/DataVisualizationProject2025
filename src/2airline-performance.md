---
theme: dashboard
title: Airline performance ✈️
toc: true
---

# Airline performance ✈️

<br>

## Overview

```js
console.log("🚀 Bar chart function is running!");

// Load dataset
const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });
console.log("🛠 Data loaded:", datasetFlights); // Debugging

// Airline Names
const airlines = [...new Set(datasetFlights.map(d => d.AIRLINE))];

// Create Toggle (Total, Canceled, Diverted) with a Default Value
const selectedView = Inputs.radio(["Total Flights", "Canceled Flights", "Diverted Flights"], {
  label: "✈ Select View",
  value: "Total Flights" // Set Default
});

// Set Dimensions
const width = 900, height = 500, margin = { top: 30, right: 40, bottom: 100, left: 70 };

// Function to Compute Flight Data
function computeFlightCounts() {
  return airlines.map(airline => {
    const flights = datasetFlights.filter(d => d.AIRLINE === airline);
    return {
      airline,
      total: flights.length,
      canceled: flights.filter(d => d.CANCELLED > 0).length,
      diverted: flights.filter(d => d.DIVERTED > 0).length
    };
  });
}

// Compute Separate Y Scales
const flightData = computeFlightCounts();
const maxTotalFlights = d3.max(flightData, d => d.total);
const maxCanceledFlights = d3.max(flightData, d => d.canceled);
const maxDivertedFlights = d3.max(flightData, d => d.diverted);

// Function to Draw Chart
function drawBarChart() {
  console.log("📊 Selected View:", selectedView.value);

  // Remove previous chart
  d3.select("#barchart-container").html("");

  // Create SVG
  const svg = d3.select("#barchart-container").append("svg")
    .attr("width", width)
    .attr("height", height);

  // X Scale (Airlines)
  const xScale = d3.scaleBand()
    .domain(flightData.map(d => d.airline))
    .range([margin.left, width - margin.right])
    .padding(0.3);

  // Y Scale (Different per selection)
  let yScale;
  if (selectedView.value === "Total Flights") {
    yScale = d3.scaleLinear()
      .domain([0, maxTotalFlights])
      .nice()
      .range([height - margin.bottom, margin.top]);
  } else if (selectedView.value === "Canceled Flights") {
    yScale = d3.scaleLinear()
      .domain([0, maxCanceledFlights])
      .nice()
      .range([height - margin.bottom, margin.top]);
  } else {
    yScale = d3.scaleLinear()
      .domain([0, maxDivertedFlights])
      .nice()
      .range([height - margin.bottom, margin.top]);
  }

  // Axes
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(xScale))
    .selectAll("text")
    .attr("transform", "rotate(-45)")
    .style("text-anchor", "end")
    .style("fill", "white");

  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(yScale));

  // Tooltip
  const tooltip = d3.select("body").append("div")
    .attr("class", "tooltip")
    .style("position", "absolute")
    .style("background", "#222")
    .style("color", "white")
    .style("padding", "8px")
    .style("border-radius", "5px")
    .style("font-size", "14px")
    .style("pointer-events", "none")
    .style("display", "none");

  if (selectedView.value === "Total Flights") {
    // **Stacked Bars for Total Flights**
    svg.append("g")
      .selectAll("g")
      .data(flightData)
      .join("g")
      .each(function(d) {
        const g = d3.select(this);
        let yPos = yScale(0);

        // Stacked Order: Diverted → Canceled → Normal
        ["diverted", "canceled", "total"].forEach((category, i) => {
          const barHeight = height - margin.bottom - yScale(d[category]);

          g.append("rect")
            .attr("x", xScale(d.airline))
            .attr("y", yPos - barHeight)
            .attr("height", barHeight)
            .attr("width", xScale.bandwidth())
            .attr("fill", ["#f1c40f", "#e74c3c", "#3498db"][i]) // Diverted (Yellow), Canceled (Red), Normal (Blue)
            .on("mouseover", function(event) {
              d3.select(this).style("opacity", 0.8);
              tooltip.style("display", "block")
                .html(`📊 ${d.airline}<br>✈ ${category}: ${d[category]}`);
            })
            .on("mousemove", event => {
              tooltip.style("top", `${event.pageY + 10}px`)
                .style("left", `${event.pageX + 10}px`);
            })
            .on("mouseout", function() {
              d3.select(this).style("opacity", 1);
              tooltip.style("display", "none");
            });

          yPos -= barHeight; // Stack next category on top
        });
      });
  } else {
    // **Single Bars for Canceled/Diverted**
    svg.append("g")
      .selectAll("rect")
      .data(flightData)
      .join("rect")
      .attr("x", d => xScale(d.airline))
      .attr("y", d => yScale(d[selectedView.value.toLowerCase().split(" ")[0]])) // Either 'canceled' or 'diverted'
      .attr("height", d => height - margin.bottom - yScale(d[selectedView.value.toLowerCase().split(" ")[0]]))
      .attr("width", xScale.bandwidth())
      .attr("fill", selectedView.value === "Canceled Flights" ? "#e74c3c" : "#f1c40f") // Red for Canceled, Yellow for Diverted
      .on("mouseover", function(event, d) {
        d3.select(this).style("opacity", 0.8);
        tooltip.style("display", "block")
          .html(`📊 ${d.airline}<br>✈ ${selectedView.value}: ${d[selectedView.value.toLowerCase().split(" ")[0]]}`);
      })
      .on("mousemove", event => {
        tooltip.style("top", `${event.pageY + 10}px`)
          .style("left", `${event.pageX + 10}px`);
      })
      .on("mouseout", function() {
        d3.select(this).style("opacity", 1);
        tooltip.style("display", "none");
      });
  }

  return svg.node();
}

// Generate Chart with a Default Value
selectedView.value = "Total Flights"; // Ensure it's not null
const barChart = drawBarChart();

// Update Chart on Toggle Change
selectedView.addEventListener("input", () => {
  d3.select("#barchart-container").html("");
  drawBarChart();
});

```

<div style="display: flex; gap: 15px;">
  <div class="filter"> ${selectedView} </div>
</div>

<div class="grid grid-cols-1">
  <div class="card">
    <div id="barchart-container"></div> <!-- This is the div where the chart will be inserted -->
  </div>
</div>

<p>

This **interactive bar chart** displays the number of flights per airline.  

✔ **Use the toggle** to switch between total flights, canceled flights, or diverted flights.  

✔ **Hover over bars** to see detailed flight stats.  

✔ **Colors differentiate** the type of flights:  
  - 🔵 **Blue** → Total Flights  
  - 🔴 **Red** → Canceled Flights  
  - 🟡 **Yellow** → Diverted Flights  
</p>


## Flight Status Flow
```js
async function createSankeyChart() {
  // Load required libraries
  const [d3, { sankey, sankeyLinkHorizontal }] = await Promise.all([
    import("https://cdn.jsdelivr.net/npm/d3@7/+esm"),
    import("https://cdn.jsdelivr.net/npm/d3-sankey@0.12/+esm")
  ]);

  // Convert date format properly
  datasetFlights.forEach(d => d.FL_DATE = new Date(d.FL_DATE));

  // Group flights by airline and flight status
  const airlineStatusCounts = d3.rollups(
    datasetFlights,
    v => ({
      "On-Time": v.filter(f => f.CANCELLED === 0 && f.DIVERTED === 0 && f.DEP_DELAY <= 15).length,
      "Delayed": v.filter(f => f.CANCELLED === 0 && f.DIVERTED === 0 && f.DEP_DELAY > 15).length,
      "Cancelled": v.filter(f => f.CANCELLED === 1).length,
      "Diverted": v.filter(f => f.DIVERTED === 1).length
    }),
    d => d.AIRLINE
  );

  // Define nodes
  const nodes = [{ name: "Total Flights" }];

  // Add airlines as nodes
  const airlineNodes = airlineStatusCounts.map(([airline]) => ({ name: airline }));
  nodes.push(...airlineNodes);

  // Add flight status categories as nodes
  const statusNodes = ["On-Time", "Delayed", "Cancelled", "Diverted"].map(name => ({ name }));
  nodes.push(...statusNodes);

  // Create index lookup
  const nodeIndexMap = Object.fromEntries(nodes.map((n, i) => [n.name, i]));

  // Create links
  const links = [];

  // Connect "Total Flights" to each airline
  airlineStatusCounts.forEach(([airline, counts]) => {
    links.push({
      source: nodeIndexMap["Total Flights"],
      target: nodeIndexMap[airline],
      value: counts["On-Time"] + counts["Delayed"] + counts["Cancelled"] + counts["Diverted"],
      category: "Total"
    });

    // Connect each airline to flight status categories
    Object.entries(counts).forEach(([status, count]) => {
      if (count > 0) {
        links.push({
          source: nodeIndexMap[airline],
          target: nodeIndexMap[status],
          value: count,
          category: status
        });
      }
    });
  });

  // **Set up dimensions**
  const width = 950, height = 650; // Adjusted to fit labels

  // **Define Sankey layout**
  const { nodes: sankeyNodes, links: sankeyLinks } = sankey()
    .nodeWidth(15)
    .nodePadding(10)
    .extent([[100, 20], [width - 100, height - 20]])({ // Centering the chart
      nodes: nodes.map(d => Object.assign({}, d)),
      links: links.map(d => Object.assign({}, d))
    });

  // **Select the container div**
  const container = d3.select("#sankey-container");

  // **Remove old SVG**
  container.select("svg").remove();

  // **Create SVG element**
  const svg = container.append("svg")
    .attr("viewBox", `0 0 ${width} ${height}`)
    .attr("width", "100%")
    .attr("height", height)
    .style("display", "block")
    .style("margin", "auto")
    .style("font", "12px sans-serif");

  // **Define color scale**
  const colorScale = d3.scaleOrdinal()
    .domain(["Total", "On-Time", "Delayed", "Cancelled", "Diverted"])
    .range(["#f39c12", "#2ecc71", "#f1c40f", "#e74c3c", "#9b59b6"]);

  // **Tooltip**
  const tooltip = d3.select("body").append("div")
    .attr("class", "tooltip")
    .style("position", "absolute")
    .style("background", "rgba(0, 0, 0, 0.9)")
    .style("color", "white")
    .style("padding", "8px 12px")
    .style("border-radius", "4px")
    .style("font-size", "12px")
    .style("pointer-events", "none")
    .style("display", "none");

  // **Draw Links**
  svg.append("g")
    .selectAll("path")
    .data(sankeyLinks)
    .join("path")
    .attr("d", sankeyLinkHorizontal())
    .attr("stroke", d => colorScale(d.category))
    .attr("stroke-width", d => Math.max(1, d.width))
    .attr("fill", "none")
    .attr("opacity", 0.6)
    .on("mouseover", function (event, d) {
      d3.select(this).attr("opacity", 1);
      tooltip.style("display", "block")
        .html(`
          <strong>${nodes[d.source.index]?.name || "Unknown"} → ${nodes[d.target.index]?.name || "Unknown"}</strong><br>
          📊 Flights: ${d.value}
        `);
    })
    .on("mousemove", event => {
      let tooltipWidth = 150;
      let xPos = event.pageX + 15;
      let yPos = event.pageY + 15;

      if (xPos + tooltipWidth > window.innerWidth) {
        xPos = event.pageX - tooltipWidth - 15;
      }
      tooltip.style("top", `${yPos}px`).style("left", `${xPos}px`);
    })
    .on("mouseout", function () {
      d3.select(this).attr("opacity", 0.6);
      tooltip.style("display", "none");
    });

  // **Draw Nodes**
  svg.append("g")
    .selectAll("rect")
    .data(sankeyNodes)
    .join("rect")
    .attr("x", d => d.x0)
    .attr("y", d => d.y0)
    .attr("height", d => d.y1 - d.y0)
    .attr("width", d => d.x1 - d.x0)
    .attr("fill", d => colorScale(d.name))
    .append("title")
    .text(d => `${d.name}`);

  // **Add Labels**
  svg.append("g")
    .selectAll("text")
    .data(sankeyNodes)
    .join("text")
    .attr("x", d => d.x0 < width / 2 ? d.x0 - 10 : d.x1 + 10)
    .attr("y", d => (d.y0 + d.y1) / 2)
    .attr("dy", "0.35em")
    .attr("text-anchor", d => d.x0 < width / 2 ? "end" : "start")
    .style("fill", "white")
    .style("font-size", "11px")
    .text(d => d.name);
}

// **Run the function to create the chart**
createSankeyChart();

```
<div class="grid grid-cols-1"> 
  <div class="card"> <div id="sankey-container"></div> </div> 
</div>


## Flight Distance vs. Average Delay per Airline

<!--
```js
console.log("🚀 Scatter plot function is running!"); 

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
  .domain([0, d3.max(scatterStats, d => d.avgDelay) || 100]) // ✅ Default value to avoid undefined
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
  .attr("opacity", 0.7)
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

// ✅ Step 7: Add Title
scatterSvg.append("text")
  .attr("x", scatterWidth / 2)
  .attr("y", 20)
  .attr("text-anchor", "middle")
  .style("font-size", "16px")
  .style("fill", "white")
  .text("Flight Distance vs. Average Delay per Airline");

```
-->

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

