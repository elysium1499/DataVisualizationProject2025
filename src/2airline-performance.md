---
theme: dashboard
title: Airline performance
toc: true
---

# Airline performance ✈️

<br>

## Overview
<br>

```js
const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });
```


```js
// Airline Names
const airlines = [...new Set(datasetFlights.map(d => d.AIRLINE))];

// Create Dropdown Menu (On-Time, Delayed, Canceled, Diverted) with a Default Value
const selectedView = Inputs.select(["Total Flights", "On-Time Flights", "Delayed Flights", "Cancelled Flights", "Diverted Flights"], {
  label: "✈ Select View",
  value: "Total Flights"
});

// Set Dimensions
const width = 900, height = 500, margin = { top: 30, right: 40, bottom: 100, left: 70 };

// Function to Compute Flight Data
function computeFlightCounts() {
  return airlines.map(airline => {
    const flights = datasetFlights.filter(d => d.AIRLINE === airline);
    const onTimeFlights = flights.filter(f => f.CANCELLED === 0 && f.DIVERTED === 0 && f.DEP_DELAY <= 15);
    const delayedFlights = flights.filter(f => f.CANCELLED === 0 && f.DIVERTED === 0 && f.DEP_DELAY > 15);
    const cancelledFlights = flights.filter(f => f.CANCELLED === 1);
    const divertedFlights = flights.filter(f => f.DIVERTED === 1);

    return {
      airline,
      onTime: onTimeFlights.length,
      delayed: delayedFlights.length,
      cancelled: cancelledFlights.length,
      diverted: divertedFlights.length,
      total: flights.length
    };
  });
}

// Compute Separate Y Scales
const flightData = computeFlightCounts();
const maxOnTimeFlights = d3.max(flightData, d => d.onTime);
const maxDelayedFlights = d3.max(flightData, d => d.delayed);
const maxCanceledFlights = d3.max(flightData, d => d.cancelled);
const maxDivertedFlights = d3.max(flightData, d => d.diverted);
const maxTotalFlights = d3.max(flightData, d => d.total);

// Function to Draw Chart
function drawBarChart() {
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
  } else if (selectedView.value === "Cancelled Flights") {
    yScale = d3.scaleLinear()
      .domain([0, maxCanceledFlights])
      .nice()
      .range([height - margin.bottom, margin.top]);
  } else if (selectedView.value === "Diverted Flights") {
    yScale = d3.scaleLinear()
      .domain([0, maxDivertedFlights])
      .nice()
      .range([height - margin.bottom, margin.top]);
  } else if (selectedView.value === "On-Time Flights") {
    yScale = d3.scaleLinear()
      .domain([0, maxOnTimeFlights])
      .nice()
      .range([height - margin.bottom, margin.top]);
  } else if (selectedView.value === "Delayed Flights") {
    yScale = d3.scaleLinear()
      .domain([0, maxDelayedFlights])
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

        // Stacked Order: Diverted → Canceled → Delayed → On-Time
        ["diverted", "cancelled", "delayed", "onTime"].forEach((category, i) => {
          const barHeight = height - margin.bottom - yScale(d[category]);
          let color;
          switch (category) {
            case "onTime": color = "#4caf50"; break; // Green
            case "delayed": color = "#ffeb3b"; break; // Yellow
            case "cancelled": color = "#f44336"; break; // Red
            case "diverted": color = "#ff9800"; break; // Orange
          }

          g.append("rect")
            .attr("x", xScale(d.airline))
            .attr("y", yPos - barHeight)
            .attr("height", barHeight)
            .attr("width", xScale.bandwidth())
            .attr("fill", color)
            .on("mouseover", function(event) {
              d3.select(this).style("opacity", 0.8);
              tooltip.style("display", "block")
                .html(`📊 ${d.airline}<br>✈ ${category.charAt(0).toUpperCase() + category.slice(1)} Flights: ${d[category]}`);
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
    // **Single Bars for On-Time, Delayed, Canceled/Diverted**
    const selectedCategory = selectedView.value.toLowerCase().split(" ")[0]; // 'on-time', 'delayed', 'cancelled', 'diverted'
    const colorMap = {
      "on-time": "#4caf50", // Green
      "delayed": "#ffeb3b", // Yellow
      "cancelled": "#f44336", // Red
      "diverted": "#9b59b6"   // Violet
    };
    const selectedDataKey = selectedCategory === "on-time" ? "onTime" : selectedCategory;

    svg.append("g")
      .selectAll("rect")
      .data(flightData)
      .join("rect")
      .attr("x", d => xScale(d.airline))
      .attr("y", d => yScale(d[selectedDataKey]))
      .attr("height", d => height - margin.bottom - yScale(d[selectedDataKey]))
      .attr("width", xScale.bandwidth())
      .attr("fill", colorMap[selectedCategory])
      .on("mouseover", function(event, d) {
        d3.select(this).style("opacity", 0.8);
        tooltip.style("display", "block")
          .html(`📊 ${d.airline}<br>✈ ${selectedView.value}: ${d[selectedDataKey]}`);
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

  // **Create Legend in Top-Right Corner**
  const legend = svg.append("g")
    .attr("transform", `translate(${width - margin.right - 180},${margin.top})`);

  // Create Legend Items
  const legendData = [
    { label: "On-Time Flights", color: "#4caf50" },   // Green
    { label: "Delayed Flights", color: "#ffeb3b" }, // Yellow
    { label: "Cancelled Flights", color: "#f44336" }, // Red
    { label: "Diverted Flights", color: "#9b59b6" }   // Violet
  ];

  // Draw Legend Items
  legendData.forEach((item, index) => {
    legend.append("rect")
      .attr("x", 0)
      .attr("y", index * 25)
      .attr("width", 20)
      .attr("height", 20)
      .attr("fill", item.color);

    legend.append("text")
      .attr("x", 30)
      .attr("y", index * 25 + 15)
      .style("fill", "white")
      .style("font-size", "14px")
      .text(item.label);
  });

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

// Append the dropdown menu to the document body or a specific container
document.body.appendChild(selectedView);
```

<div style="font-family: 'Times New Roman', serif;">
  <div style="display: inline-block; width: 250px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color:#292929; color: white;">${selectedView}</div>
  <div class="grid grid-cols-1"> <div class="card"><div id="barchart-container"></div> </div> </div>
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

<br>

## Flight Status Flow
<br>

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
  const width = 900, height =600; // Increased height to move the chart down

  // **Define Sankey layout**
  const { nodes: sankeyNodes, links: sankeyLinks } = sankey()
    .nodeWidth(20)
    .nodePadding(5)
    .extent([[100, 50], [width - 100, height - 20]])({ // Increased vertical padding for more space
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
    .style("margin", "0 auto") // Center the entire SVG
    .style("font", "12px sans-serif");

  // **Define color scales**
  const colorA = "#f39c12"; // Color A (Total Flights + Links)
  const colorB = "#3498db"; // Color B (Airlines)
  const colorScaleStatus = d3.scaleOrdinal()
    .domain(["On-Time", "Delayed", "Cancelled", "Diverted"])
    .range(["#2ecc71", "#f1c40f", "#e74c3c", "#9b59b6"]); // Colors C, D, E, F for statuses

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
    .attr("stroke", d => {
      if (nodes[d.source.index].name === "Total Flights") {
        return colorA; // Color A for Total Flights and its links
      } else {
        return colorScaleStatus(d.category); // Color for the status links (C, D, E, F)
      }
    })
    .attr("stroke-width", d => Math.max(1, d.width))
    .attr("fill", "none")
    .attr("opacity", 0.6)
    .on("mouseover", function (event, d) {
      d3.select(this).attr("opacity", 1);
      tooltip.style("display", "block")
        .html(
          `<strong>${nodes[d.source.index]?.name || "Unknown"} → ${nodes[d.target.index]?.name || "Unknown"}</strong><br>📊 Flights: ${d.value}`
        );
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
    .attr("fill", d => {
      if (d.name === "Total Flights") {
        return colorA; // Total Flights bar color
      } else if (airlineStatusCounts.some(([airline]) => airline === d.name)) {
        return colorB; // Airline bar color
      } else {
        return colorScaleStatus(d.name); // Status bar colors (C, D, E, F)
      }
    })
    .append("title")
    .text(d => `${d.name}`);

  // **Add Labels**
  svg.append("g")
    .selectAll("text")
    .data(sankeyNodes)
    .join("text")
    .attr("x", d => d.x0 < width / 2 ? d.x0 - 10 : d.x1 + 10)
    .attr("y", d => (d.y0 + d.y1) / 2)
    .attr("dy", "0.40em")
    .attr("text-anchor", d => d.x0 < width / 2 ? "end" : "start")
    .style("fill", "white")
    .style("font-size", "11px")
    .text(d => d.name);

  // **Add Column Headers Above the Sankey Diagram**
  const headerHeight = 30;  // Set a height for the header labels

  // Add "Number of Flights" header above the "Total Flights" node
  svg.append("text")
    .attr("x", width / 8)
    .attr("y", 25)  // Position at the top
    .attr("dy", headerHeight / 2)  // Center vertically
    .attr("text-anchor", "middle")
    .style("font-size", "14px")
    .style("fill", "white")
    .text("Number of Flights");

  // Add "Airport" header above the airline nodes
  const airportWidth = 100; // Adjust the width according to your layout
  const airportXPos = width / 2; // Adjust this based on your Sankey layout
  svg.append("text")
    .attr("x", airportXPos)
    .attr("y", 25)  // Position at the top
    .attr("dy", headerHeight / 2)  // Center vertically
    .attr("text-anchor", "middle")
    .style("font-size", "14px")
    .style("fill", "white")
    .text("Airline");

  // Add "Status" header above the status nodes (On-Time, Delayed, Cancelled, Diverted)
  const statusXPos = width;  // You might want to adjust this if status nodes are spread out horizontally
  svg.append("text")
    .attr("x", statusXPos-100)
    .attr("y", 25)  // Position at the top
    .attr("dy", headerHeight / 2)  // Center vertically
    .attr("text-anchor", "middle")
    .style("font-size", "14px")
    .style("fill", "white")
    .text("Status");
}

createSankeyChart();

```
<div class="grid grid-cols-1"> 
  <div class="card"> <div id="sankey-container"></div> </div> 
</div>

