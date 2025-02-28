---
theme: dashboard
title: Global Trends 🌍
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

# Global Trends 🌍
<br>

## Flight Volume Over Time 📈
```js
const stateNameMap = {
  "AL": "Alabama", "AK": "Alaska", "AZ": "Arizona", "AR": "Arkansas",
  "CA": "California", "CO": "Colorado", "CT": "Connecticut", "DE": "Delaware",
  "FL": "Florida", "GA": "Georgia", "HI": "Hawaii", "ID": "Idaho",
  "IL": "Illinois", "IN": "Indiana", "IA": "Iowa", "KS": "Kansas",
  "KY": "Kentucky", "LA": "Louisiana", "ME": "Maine", "MD": "Maryland",
  "MA": "Massachusetts", "MI": "Michigan", "MN": "Minnesota", "MS": "Mississippi",
  "MO": "Missouri", "MT": "Montana", "NE": "Nebraska", "NV": "Nevada",
  "NH": "New Hampshire", "NJ": "New Jersey", "NM": "New Mexico", "NY": "New York",
  "NC": "North Carolina", "ND": "North Dakota", "OH": "Ohio", "OK": "Oklahoma",
  "OR": "Oregon", "PA": "Pennsylvania", "RI": "Rhode Island", "SC": "South Carolina",
  "SD": "South Dakota", "TN": "Tennessee", "TX": "Texas", "UT": "Utah",
  "VT": "Vermont", "VA": "Virginia", "WA": "Washington", "WV": "West Virginia",
  "WI": "Wisconsin", "WY": "Wyoming",  "DC": "District of Columbia", "PR": "Puerto Rico", "VI": "U.S. Virgin Islands", "TT": "Trust Territory of the Pacific Islands"
};

// Set up dimensions
const width = 1000;
const height = 500;
const margin = { top: 30, right: 40, bottom: 50, left: 70 };

// Create dropdowns and toggle for filtering
const airlineOptions = ["All Airlines", ...new Set(datasetFlights.map(d => d.AIRLINE))];
//const destinationOptions = ["All Destinations", ...new Set(datasetFlights.map(d => d.DEST_STATE))];
const destinationOptions = ["All Destinations", ...new Set(datasetFlights.map(d => stateNameMap[d.DEST_STATE] || d.DEST_STATE))];

const selectedAirline = Inputs.select(airlineOptions, { label: "✈ Select Airline" });
const selectedDestination = Inputs.select(destinationOptions, { label: "📍 Select Destination" });
const smoothLine = Inputs.toggle({ label: "📈 Smooth Line" });

// Function to filter and aggregate data
function getFilteredData(airline, destination) {
  return d3.rollups(
    datasetFlights.filter(d =>
      (airline === "All Airlines" || d.AIRLINE === airline) &&
      //(destination === "All Destinations" || d.DEST_STATE === destination)
      (destination === "All Destinations" || stateNameMap[d.DEST_STATE] === destination)
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















## Map

```js
// Import necessary D3 libraries
//const d3 = await import("https://cdn.jsdelivr.net/npm/d3@7/+esm");
const topojson = await import("https://cdn.jsdelivr.net/npm/topojson@3/+esm");

// Load dataset
//const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });

// Load US states GeoJSON
const usStates = await d3.json("https://cdn.jsdelivr.net/npm/us-atlas@3/states-10m.json");

// Convert date format properly
datasetFlights.forEach(d => d.FL_DATE = new Date(d.FL_DATE));

// Convert state abbreviations to full names
const stateAbbreviations = {
  "AL": "Alabama", "AK": "Alaska", "AZ": "Arizona", "AR": "Arkansas", "CA": "California",
  "CO": "Colorado", "CT": "Connecticut", "DE": "Delaware", "FL": "Florida", "GA": "Georgia",
  "HI": "Hawaii", "ID": "Idaho", "IL": "Illinois", "IN": "Indiana", "IA": "Iowa",
  "KS": "Kansas", "KY": "Kentucky", "LA": "Louisiana", "ME": "Maine", "MD": "Maryland",
  "MA": "Massachusetts", "MI": "Michigan", "MN": "Minnesota", "MS": "Mississippi",
  "MO": "Missouri", "MT": "Montana", "NE": "Nebraska", "NV": "Nevada", "NH": "New Hampshire",
  "NJ": "New Jersey", "NM": "New Mexico", "NY": "New York", "NC": "North Carolina",
  "ND": "North Dakota", "OH": "Ohio", "OK": "Oklahoma", "OR": "Oregon", "PA": "Pennsylvania",
  "RI": "Rhode Island", "SC": "South Carolina", "SD": "South Dakota", "TN": "Tennessee",
  "TX": "Texas", "UT": "Utah", "VT": "Vermont", "VA": "Virginia", "WA": "Washington",
  "WV": "West Virginia", "WI": "Wisconsin", "WY": "Wyoming", "DC": "District of Columbia", "PR": "Puerto Rico", "VI": "U.S. Virgin Islands", "TT": "Trust Territory of the Pacific Islands"
};

// Convert state abbreviations to full names
datasetFlights.forEach(d => {
  d.ORIGIN_STATE = stateAbbreviations[d.ORIGIN_STATE] || d.ORIGIN_STATE;
});

// Extract available years
const availableYears = [...new Set(datasetFlights.map(d => d.FL_DATE.getFullYear()))]
  .sort((a, b) => a - b)
  .map(year => String(year)); // Convert to string for dropdown

// Create dropdowns for filtering
const selectedYear = Inputs.select(availableYears, { label: "📅 Select Year" });
const airlineOptions = ["All Airlines", ...new Set(datasetFlights.map(d => d.AIRLINE))];
const selectedAirline1 = Inputs.select(airlineOptions, { label: "✈ Select Airline" });

// Function to filter flights by year and airline
function filterFlights(year, airline) {
  return datasetFlights.filter(d =>
    d.FL_DATE.getFullYear() === Number(year) &&
    (airline === "All Airlines" || d.AIRLINE === airline)
  );
}

// Function to compute flight counts per state
function computeStateFlightCounts(data) {
  const counts = d3.rollups(
    data,
    v => v.length, // Count flights per state
    d => d.ORIGIN_STATE
  );
  return Object.fromEntries(counts);
}

// Define map dimensions
const width = 900, height = 600;

// Projection & path generator
const projection = d3.geoAlbersUsa().fitSize([width, height], topojson.feature(usStates, usStates.objects.states));
const path = d3.geoPath().projection(projection);

// Define color scales
// Compute dynamic quantile thresholds based on dataset
function computeColorScale(data, colorRange) {
  const stateCounts = Object.values(computeStateFlightCounts(data));
  return d3.scaleQuantile()
    .domain(stateCounts)
    .range(colorRange);
}

// Function to draw the map with quantile-based color scaling
function drawMap(data) {
  const stateCounts = computeStateFlightCounts(data);
  const isAllAirlines = selectedAirline1.value === "All Airlines";

  // Choose color range
  const colorRange = isAllAirlines ? d3.schemeBlues[5] : d3.schemeOranges[5];

  // Compute quantile color scale dynamically
  const colorScale = computeColorScale(data, colorRange);

  // Join state data with flight counts
  const statesWithCounts = topojson.feature(usStates, usStates.objects.states).features.map(d => {
    const stateName = d.properties.name;
    d.properties.flights = stateCounts[stateName] || 0;
    return d;
  });

  // Clear previous map
  d3.select("#map-container").html("");

  // Create SVG element
  const svg = d3.select("#map-container").append("svg")
    .attr("width", width)
    .attr("height", height);

  // Tooltip
  const tooltip = d3.select("body").append("div")
    .attr("class", "tooltip")
    .style("position", "absolute")
    .style("background", "rgba(0, 0, 0, 0.7)")
    .style("color", "white")
    .style("padding", "5px 10px")
    .style("border-radius", "4px")
    .style("font-size", "12px")
    .style("pointer-events", "none")
    .style("display", "none");

  // Draw states
  svg.append("g")
    .selectAll("path")
    .data(statesWithCounts)
    .join("path")
    .attr("d", path)
    .attr("fill", d => colorScale(d.properties.flights))
    .attr("stroke", "#222")
    .on("mouseover", function (event, d) {
      d3.select(this).attr("stroke", "white");
      tooltip.style("display", "block")
        .html(`<strong>${d.properties.name}</strong><br>Flights: ${d.properties.flights}`);
    })
    .on("mousemove", event => {
      tooltip.style("top", `${event.pageY + 10}px`)
        .style("left", `${event.pageX + 10}px`);
    })
    .on("mouseout", function () {
      d3.select(this).attr("stroke", "#222");
      tooltip.style("display", "none");
    });

  // Add legend for quantile scale
  // Safely extract quantile breakpoints and ensure it's always an array
  // Extract quantile breakpoints safely// Extract quantile breakpoints safely
  // Extract quantile breakpoints safely
  const quantiles = Array.isArray(colorScale.quantiles()) ? colorScale.quantiles() : [];

  // Ensure stateCounts is an array before using Math.min() & Math.max()
  const minCount = stateCounts.length > 0 ? Math.min(...stateCounts) : 0;
  const maxCount = stateCounts.length > 0 ? Math.max(...stateCounts) : 0;

  // Create bin ranges from quantiles
  const legendRanges = [minCount, ...quantiles.map(d => Math.round(d)), maxCount];

  // Generate range labels (e.g., "0-21", "22-36")
  //const legendLabels = legendRanges.slice(0, -1).map((d, i) => `${d+1}-${legendRanges[i + 1]}`);
  //const legendLabels = legendRanges.slice(0, -1).map((d, i) => {
  //const nextValue = i < legendRanges.length - 2 ? legendRanges[i + 1] - 1 : legendRanges[i + 1]; 
  //return `${d}-${nextValue}`;});

  if (legendRanges[legendRanges.length - 1] < maxCount) {
  legendRanges.push(maxCount);}

  // Generate range labels ensuring "+1 condition"
  const legendLabels = legendRanges.slice(0, -1).map((d, i) => {
    let nextValue = legendRanges[i + 1] - 1;
    //if (i === legendRanges.length - 2) nextValue = legendRanges[i + 1]; // Ensure last bin fully includes maxCount
    if (i === legendRanges.length - 2) return `${d} -`; // Ensure last bin fully includes maxCount
    return `${d}-${nextValue}`;
  });

  // Create legend
  const legend = svg.append("g")
    .attr("transform", `translate(${width - 150}, 20)`);

  // Draw color boxes
  legend.selectAll("rect")
    .data(colorScale.range()) // Use quantile colors
    .join("rect")
    .attr("x", 0)
    .attr("y", (d, i) => i * 20)
    .attr("width", 20)
    .attr("height", 20)
    .attr("fill", d => d);

  // Draw text labels for ranges
  legend.selectAll("text")
    .data(legendLabels)
    .join("text")
    .attr("x", 30)
    .attr("y", (d, i) => i * 20 + 15)
    .attr("font-size", "12px")
    .attr("fill", "white")
    .text(d => d);


}

// Update map when filters change
function updateMap() {
  const year = selectedYear.value;
  const airline = selectedAirline1.value;
  const filteredData = filterFlights(year, airline);

  if (filteredData.length === 0) {
    console.warn("⚠ No flight data available for", year, airline);
    d3.select("#map-container").html("<p style='color:red;'>No data available.</p>");
    return;
  }

  drawMap(filteredData);
}

// Listen for dropdown changes
selectedYear.addEventListener("input", updateMap);
selectedAirline1.addEventListener("input", updateMap);

// Initial draw
updateMap();


```
<div style="display: flex; gap: 15px;"> <div class="filter"> ${selectedYear} </div> <div class="filter"> ${selectedAirline1} </div> </div> <div class="grid grid-cols-1"> <div class="card"> <div id="map-container"></div> </div> </div> 

<p> 

This **interactive flight map** allows you to explore **USA flight routes** by: 

🔹 **Selecting a year** to filter flight data. 

🔹 **Choosing an airline** to visualize specific airline coverage. 

🔹 **Viewing flight density** with states shaded according to flight volume. 

🔹 **Hovering over a state** to see the exact number of flights.

 </p> 