---
theme: dashboard
title: Global Trends 
toc: true
---

```js
// Load dataset
const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });

// Convert date format properly
datasetFlights.forEach(d => d.FL_DATE = new Date(d.FL_DATE));

// Define COVID-19 start date
const covidStartDate = new Date("2020-03-01");
```

# Global Trends copy
<br>

## Flight Volume Over Time 📈
```js

// Set up dimensions
const width = 1000;
const height = 500;
const margin = { top: 30, right: 40, bottom: 50, left: 70 };

// Create dropdowns and toggle for filtering
const airlineOptions = ["All Airlines", ...new Set(datasetFlights.map(d => d.AIRLINE))];
const destinationOptions = ["All Destinations", ...new Set(datasetFlights.map(d => d.DEST_CITY_NAME))];

const selectedAirline = Inputs.select(airlineOptions, { label: "✈ Select Airline" });
const selectedDestination = Inputs.select(destinationOptions, { label: "📍 Select Destination" });
const smoothLine = Inputs.toggle({ label: "📈 Smooth Line" });

// Function to filter and aggregate data
function getFilteredData(airline, destination) {
  return d3.rollups(
    datasetFlights.filter(d =>
      (airline === "All Airlines" || d.AIRLINE === airline) &&
      (destination === "All Destinations" || d.DEST_CITY_NAME === destination)
    ),
    v => v.length,
    d => d.FL_DATE
  ).map(([date, count]) => ({ date, count }))
   .sort((a, b) => a.date - b.date);
}

  // Aggregate flights per day
const flightsPerDay = d3.rollups(
  datasetFlights,
  v => v.length, // Count flights per day
  d => new Date(d.FL_DATE)
).map(([date, count]) => ({ date, count }))
 .sort((a, b) => a.date - b.date);


// Function to draw the chart
function drawChart() {
  const airline = selectedAirline.value;
  const destination = selectedDestination.value;
  const smooth = smoothLine.value;

  const filteredData = getFilteredData(airline, destination);

  // Create scales
  const xScale = d3.scaleTime()
    .domain(d3.extent(filteredData, d => d.date))
    .range([margin.left, width - margin.right]);

  const yScale = d3.scaleLinear()
    .domain([0, d3.max(filteredData, d => d.count)])
    .nice()
    .range([height - margin.bottom, margin.top]);

  // Create line generator
  const lineGenerator = d3.line()
    .x(d => xScale(d.date))
    .y(d => yScale(d.count))
    .curve(smooth ? d3.curveBasis : d3.curveLinear); // Toggle smooth line

  // Create SVG container
  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height);

  // Add axes
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(xScale));

  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(yScale));
  

  // Add X Axis Label (Bottom)
  svg.append("text")
    .attr("x", width / 2) // Centered
    .attr("y", height - 10) // Below X axis
    .attr("fill", "white")
    .attr("text-anchor", "middle")
    .attr("font-size", "14px")
    .text("Date");

// Add Y Axis Label (Rotated)
  svg.append("text")
    .attr("x", -height / 2) // Centered along Y axis
    .attr("y", 20) // Left of Y axis
    .attr("fill", "white")
    .attr("text-anchor", "middle")
    .attr("font-size", "14px")
    .attr("transform", "rotate(-90)") // Rotate for vertical text
    .text("Number of Flights");

  // Draw line chart
  svg.append("path")
    .datum(filteredData)
    .attr("fill", "none")
    .attr("stroke", "#3498db")
    .attr("stroke-width", 2)
    .attr("d", lineGenerator);

    // Create Tooltip Div (Hidden by Default)
  const tooltip = d3.select("body").append("div")
    .attr("class", "tooltip")
    .style("position", "absolute")
    .style("background", "#222")
    .style("color", "#fff")
    .style("padding", "8px")
    .style("border-radius", "5px")
    .style("font-size", "14px")
    .style("pointer-events", "none")
    .style("display", "none");

  // Create an overlay for hover interaction
  const overlay = svg.append("rect")
    .attr("width", width)
    .attr("height", height)
    .attr("fill", "transparent")
    .on("mousemove", function(event) {
      const [mouseX] = d3.pointer(event);
      
      // Find the closest date
      const dateScale = xScale.invert(mouseX);
      const closestPoint = flightsPerDay.reduce((prev, curr) => 
        Math.abs(curr.date - dateScale) < Math.abs(prev.date - dateScale) ? curr : prev
      );

      tooltip.style("display", "block")
        .html(`📅 ${closestPoint.date.toDateString()}<br>✈ Flights: ${closestPoint.count}`)
        .style("top", (event.pageY - 10) + "px")
        .style("left", (event.pageX + 10) + "px");
    })
    .on("mouseout", function() {
      tooltip.style("display", "none");
    });




  // Add COVID-19 dashed line
  svg.append("line")
    .attr("x1", xScale(covidStartDate))
    .attr("x2", xScale(covidStartDate))
    .attr("y1", margin.top)
    .attr("y2", height - margin.bottom)
    .attr("stroke", "red")
    .attr("stroke-dasharray", "5,5")
    .attr("stroke-width", 2);

  // Add COVID-19 label
  svg.append("text")
    .attr("x", xScale(covidStartDate))
    .attr("y", margin.top)
    .attr("fill", "red")
    .attr("text-anchor", "start")
    .attr("font-size", "12px")
    .text("COVID-19 Starts");

  return svg.node();
}

// Generate chart with interactivity
const flightVolumeChart = drawChart();

// Update chart when filters change
selectedAirline.addEventListener("input", () => {
  document.querySelector("#chart-container").innerHTML = "";
  document.querySelector("#chart-container").appendChild(drawChart());
});

selectedDestination.addEventListener("input", () => {
  document.querySelector("#chart-container").innerHTML = "";
  document.querySelector("#chart-container").appendChild(drawChart());
});

smoothLine.addEventListener("input", () => {
  document.querySelector("#chart-container").innerHTML = "";
  document.querySelector("#chart-container").appendChild(drawChart());
});

```

<div style="display: flex; gap: 15px;"> <div class="filter"> ${selectedAirline} </div> <div class="filter"> ${selectedDestination} </div> <div class="filter"> ${smoothLine} </div> </div> <div class="grid grid-cols-1"> <div class="card" id="chart-container"> ${flightVolumeChart} </div> </div> 

<p> 

The **blue line** shows the number of flights per day. Use the **filters** above to select a specific **airline** or **destination**. The **red dashed line** marks the start of **COVID-19 (March 2020)**. Toggle **Smooth Line** to adjust visualization. </p> 

<br>

## State-to-State Flight Volumes 🌎

```js
// Import necessary D3 libraries
const [d3, { sankey, sankeyLinkHorizontal }] = await Promise.all([
  import("https://cdn.jsdelivr.net/npm/d3@7/+esm"),
  import("https://cdn.jsdelivr.net/npm/d3-sankey@0.12/+esm")
]);

// Load CSV data
async function loadSankeyData() {
  const data = await FileAttachment("data/sankey_data.csv").csv({ typed: true });

  // Convert flight count to integers and remove circular links (self-connections)
  return data
    .map(d => ({
      source: d.ORIGIN_STATE.trim(),
      target: d.DEST_STATE.trim(),
      value: +d.FLIGHT_COUNT
    }))
    .filter(d => d.source !== d.target); // 🚀 Remove circular links
}

// Function to create Sankey chart
async function createSankeyChart() {
  const dataset = await loadSankeyData();

  if (!dataset.length) {
    console.error("🚨 Sankey Diagram Error: No valid data available!");
    return;
  }

  // Extract unique nodes (states)
  const states = Array.from(new Set(dataset.flatMap(d => [d.source, d.target])))
                      .map(name => ({ name }));

  // Create links (routes)
  const links = dataset.map(d => ({
    source: states.findIndex(n => n.name === d.source),
    target: states.findIndex(n => n.name === d.target),
    value: d.value
  }));

  // Ensure valid Sankey input
  if (states.length === 0 || links.length === 0) {
    console.error("🚨 Sankey Diagram Error: No valid nodes or links found!");
    return;
  }

  // Set up dimensions
  const width = 1000, height = 600;

  // Define Sankey layout
  const { nodes: sankeyNodes, links: sankeyLinks } = sankey()
    .nodeWidth(20)
    .nodePadding(10)
    .extent([[1, 1], [width - 1, height - 1]])({
      nodes: states.map(d => Object.assign({}, d)),
      links: links.map(d => Object.assign({}, d))
    });

  // Select the container div
  const container = d3.select("#sankey-container");

  // Remove old SVG to prevent duplication
  container.select("svg").remove();

  // Create SVG element
  const svg = container.append("svg")
    .attr("viewBox", [0, 0, width, height])
    .attr("width", width)
    .attr("height", height)
    .style("font", "10px sans-serif");

  // Define color scale
  const color = d3.scaleOrdinal(d3.schemeCategory10);

  // Draw links (routes)
  svg.append("g")
    .selectAll("path")
    .data(sankeyLinks)
    .join("path")
    .attr("d", sankeyLinkHorizontal())
    .attr("stroke", d => color(d.source.name))
    .attr("stroke-width", d => Math.max(1, d.width))
    .attr("fill", "none")
    .attr("opacity", 0.5)
    .on("mouseover", function(event, d) {
      d3.select(this).attr("opacity", 0.8);
      tooltip.style("display", "block")
        .html(`<strong>${d.source.name} → ${d.target.name}: ${d.value} Flights</strong>`);
    })
    .on("mousemove", event => {
      tooltip.style("top", `${event.pageY + 10}px`)
        .style("left", `${event.pageX + 10}px`);
    })
    .on("mouseout", function() {
      d3.select(this).attr("opacity", 0.5);
      tooltip.style("display", "none");
    });

  // Draw nodes (states)
  svg.append("g")
    .selectAll("rect")
    .data(sankeyNodes)
    .join("rect")
    .attr("x", d => d.x0)
    .attr("y", d => d.y0)
    .attr("height", d => d.y1 - d.y0)
    .attr("width", d => d.x1 - d.x0)
    .attr("fill", d => color(d.name))
    .append("title")
    .text(d => `${d.name}`);

  // Add labels
  svg.append("g")
    .selectAll("text")
    .data(sankeyNodes)
    .join("text")
    .attr("x", d => d.x0 < width / 2 ? d.x0 - 6 : d.x1 + 6)
    .attr("y", d => (d.y0 + d.y1) / 2)
    .attr("dy", "0.35em")
    .attr("text-anchor", d => d.x0 < width / 2 ? "end" : "start")
    .style("fill", "#fff")
    .style("font-size", "12px")
    .text(d => d.name);

  // Tooltip for interactivity
  const tooltip = d3.select("body").append("div")
    .attr("class", "tooltip")
    .style("position", "absolute")
    .style("background", "#222")
    .style("color", "#fff")
    .style("padding", "8px")
    .style("border-radius", "5px")
    .style("font-size", "14px")
    .style("pointer-events", "none")
    .style("display", "none");
}

// Run the function to create the Sankey chart
createSankeyChart();
```
<div class="grid grid-cols-1">
  <div class="card">
    <div id="sankey-container"></div>
  </div>
</div>


<p>

 This **Sankey diagram** visualizes the **state-to-state flight volumes** in the U.S. 🔹 **The width of the links represents the number of flights.** 🔹 **Each node represents a state.** 🔹 **Hover over nodes to explore connections.** </p>



 ## Monthly Flight Volume 📈 

 ```js
 
// Extract unique years and properly format them
const years = [...new Set(datasetFlights.map(d => new Date(d.FL_DATE).getFullYear()))]
  .sort((a, b) => a - b)
  .map(year => year.toString()); // Ensure it's treated as a string without commas

// Dropdown for year selection
const selectedYear = Inputs.select(years, { label: "📅 Select Year" });

// Process data: Aggregate monthly flight volumes per year
function getMonthlyData(year) {
  const filteredData = datasetFlights.filter(d => new Date(d.FL_DATE).getFullYear() == year);
  const aggregated = d3.rollups(
    filteredData,
    v => v.length, // Count flights per month
    d => new Date(d.FL_DATE).getMonth()
  ).map(([month, count]) => ({ month, count }));

  // Ensure data for all 12 months exists, filling missing months with zero flights
  const monthlyCounts = Array(12).fill(0);
  aggregated.forEach(d => { monthlyCounts[d.month] = d.count; });
  return monthlyCounts;
}

// Set a fixed scale for all years with consistent tick marks
const maxFlights = 2500; // Define a static upper limit
const fixedScale = d3.scaleLinear().domain([0, maxFlights]).range([0, 250]);

// Radar chart function
function radarChart(year, { width = 600, height = 600 } = {}) {
  const data = getMonthlyData(year);
  const months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];

  // Convert data to match months
  const radius = Math.min(width, height) / 2 - 50;
  const angleSlice = (Math.PI * 2) / 12;
  const rScale = d3.scaleLinear().domain([0, maxFlights]).range([0, radius]);

  // Create SVG
  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .style("font", "14px sans-serif")
    .style("display", "block")
    .style("margin", "auto");

  const g = svg.append("g")
    .attr("transform", `translate(${width / 2}, ${height / 2})`);

  // Circular grid lines (fixed scale)
  const tickValues = [500, 1000, 1500, 2000, 2500]; // Set exact scale values
  tickValues.forEach((tick) => {
    const gridRadius = rScale(tick);
    g.append("circle")
      .attr("r", gridRadius)
      .attr("fill", "none")
      .attr("stroke", "#777")
      .attr("stroke-dasharray", "3,3");

    g.append("text")
      .attr("x", 5)
      .attr("y", -gridRadius)
      .attr("fill", "white")
      .attr("text-anchor", "middle")
      .text(tick);
  });

  // Month labels (lightened for visibility)
  months.forEach((month, i) => {
    const angle = angleSlice * i - Math.PI / 2;
    const x = Math.cos(angle) * (radius + 20);
    const y = Math.sin(angle) * (radius + 20);
    g.append("text")
      .attr("x", x)
      .attr("y", y)
      .attr("fill", "#fff") // Light color for visibility
      .attr("text-anchor", "middle")
      .attr("dy", "0.35em")
      .text(month);
  });

  // Create tooltip
  const tooltip = d3.select("body").append("div")
    .style("position", "absolute")
    .style("background", "#222")
    .style("color", "#fff")
    .style("padding", "5px")
    .style("border-radius", "5px")
    .style("font-size", "12px")
    .style("display", "none");

  // Radar area (filled)
  const areaPath = d3.lineRadial()
    .angle((_, i) => angleSlice * i)
    .radius(d => rScale(d))
    .curve(d3.curveCardinalClosed);

  g.append("path")
    .datum(data)
    .attr("d", areaPath)
    .attr("fill", "steelblue")
    .attr("opacity", 0.5)
    .attr("stroke", "blue")
    .attr("stroke-width", 2);

  // Add interactive dots
  g.selectAll(".dot")
    .data(data)
    .enter()
    .append("circle")
    .attr("class", "dot")
    .attr("cx", (d, i) => Math.cos(angleSlice * i - Math.PI / 2) * rScale(d))
    .attr("cy", (d, i) => Math.sin(angleSlice * i - Math.PI / 2) * rScale(d))
    .attr("r", 4)
    .attr("fill", "white")
    .attr("stroke", "blue")
    .on("mouseover", function(event, d) {
      tooltip.style("display", "block")
        .html(`📅 ${months[data.findIndex(e => e === d)]}<br>✈ Flights: ${d}`)
        .style("top", (event.pageY - 10) + "px")
        .style("left", (event.pageX + 10) + "px");
    })
    .on("mouseout", () => tooltip.style("display", "none"));

  return svg.node();
}

// Generate initial chart
const ridgelineChart = radarChart(years[0]);

// Update chart when year is selected
selectedYear.addEventListener("input", () => {
  document.querySelector("#ridgeline-container").innerHTML = "";
  document.querySelector("#ridgeline-container").appendChild(radarChart(selectedYear.value));
});


```
<div class="filter"> ${selectedYear} </div>
<div class="grid grid-cols-1">
  <div class="card" id="ridgeline-container"> ${ridgelineChart} </div>
</div>


<p>

📊 This **radar chart** visualizes **monthly flight volumes** in a circular layout. 
- **Hover over points** to see flight counts. 
- **Select a year** to compare trends over time.
</p>