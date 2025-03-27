---
theme: dark
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



# Global Trends 🌍
<br>

<div>
The aviation industry is a dynamic indicator of economic activity, global connectivity, and societal change. Through this dashboard, we dive into the story told by U.S. domestic flight data between 2019 and 2023. These interactive visualizations allow us not only to observe how flight volumes evolved over time but also to explore how different regions and airlines experienced these shifts. From the sharp drop during the early days of COVID-19 to the gradual resurgence in air traffic, each plot unpacks a different layer of this complex system.
</div>

<br>
<br>

## Flight Volume Over Time
<br>

<div> 
The first chart presents a time-series line graph that illustrates the number of flights per day over the span of several years. What makes this chart immediately striking is the visible collapse in flight volumes around March–April 2020—corresponding precisely to the global onset of the COVID-19 pandemic. The red dashed line offers a clear temporal anchor, making it easy to contextualize the dramatic dip.

As we progress past this low point, the chart reveals a slow but steady recovery in flight activity. Users can toggle between a smooth and linear line to better discern seasonal fluctuations or general trends. What’s particularly useful is the ability to filter by airline and destination, allowing the viewer to zoom in on how specific carriers or cities were impacted. Note that the scale adjusts dynamically when filtering.
</div>

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
const width = 900 * 0.90;
const height = 400 * 0.90;
const margin = { top: 30, right: 40, bottom: 50, left: 70 };

// Create dropdowns and toggle for filtering
const airlineOptions = ["All Airlines", ...new Set(datasetFlights.map(d => d.AIRLINE))];
const destinationOptions = ["All Destinations", ...new Set(datasetFlights.map(d => stateNameMap[d.DEST_STATE] || d.DEST_STATE))];

const selectedAirline = Inputs.select(airlineOptions, { label: "✈ Airline" });
const selectedDestination = Inputs.select(destinationOptions, { label: "📍 Destination" });
const smoothLine = Inputs.toggle({ label: "📈 Smooth Line" });

// Function to filter and aggregate data
function getFilteredData(airline, destination) {
  return d3.rollups(
    datasetFlights.filter(d =>
      (airline === "All Airlines" || d.AIRLINE === airline) &&
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
        .html(`📅 ${closestPoint.date.toDateString()}`)
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
<br>
<div style="font-family: 'Times New Roman', serif;">
  <div style="display: flex; justify-content: center; align-items: center;">
    <div style="display: inline-block; width: 300px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color:#292929; color: white;">${selectedAirline}</div>
    <div style="display: inline-block; width: 300px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color: #292929; color: white;">${selectedDestination}</div>
    <div style="display: inline-block; width: 150px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color: #292929; color: white;">${smoothLine}</div>
  </div>
  <div class="grid grid-cols-1"> 
    <div class="card" style="display: flex; justify-content: center; align-items: center; height: 400px;" id="chart-container">${flightVolumeChart}</div> 
  </div>
</div>


<div>
The chart allows for filtering by airline and destination state, so we could potentially identify if air travel increases in certain months or if specific airlines dominate particular destinations by applying those filters
The blue line shows the number of flights per day. The filters can be use to select a specific airline or destination. The red dashed line marks the start of COVID-19 (March 2020) and the toggle Smooth Line can adjust visualization. </div> 

<br>
<br>


## Destination density Map
<br>

<div> 
This is a geographical view of flight density per state, filtered by year and airline, helps identify the busiest regions. Flight volume fluctuates over time and it is influenced by different conditions: seasons, airline operations, and passenger demand.
</div> 

```js
const topojson = await import("https://cdn.jsdelivr.net/npm/topojson@3/+esm");
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

const years = [...new Set(datasetFlights.map(d => new Date(d.FL_DATE).getFullYear()))]
  .sort((a, b) => a - b)
  .map(year => year.toString()); // Ensure it's treated as a string without commas

// Dropdown for year selection
const selectedYear = Inputs.select(years, { label: "📅 Select Year" });

// Create dropdowns for filtering
const airlineOptions = ["All Airlines", ...new Set(datasetFlights.map(d => d.AIRLINE))];
const selectedAirline1 = Inputs.select(airlineOptions, { label: "✈  Select Airline" });

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
const width = 750, height = 450;

// Projection & path generator
const projection = d3.geoAlbersUsa().fitSize([width, height], topojson.feature(usStates, usStates.objects.states));
const path = d3.geoPath().projection(projection);

// Define color scales. Compute dynamic quantile thresholds based on dataset
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

  // Draw states with darker color
  svg.append("g")
    .selectAll("path")
    .data(statesWithCounts)
    .join("path")
    .attr("d", path)
    .attr("fill", d => d3.rgb(colorScale(d.properties.flights)).darker(0.5))
    .attr("stroke", "#222")
    .on("mouseover", function (event, d) {
      const [x, y] = path.centroid(d); // Calcola il centro dello stato

      d3.select(this)
        .attr("fill", d => d3.rgb(colorScale(d.properties.flights)).brighter(0.5)) // Make it brighter
        .attr("stroke", "white") // Change stroke color
        .transition().duration(200) // Smooth transition
        .attr("transform", `translate(${x * -0.3}, ${y * -0.3}) scale(1.3)`);

      // Sposta il path sopra agli altri
      d3.select(this).raise(); // Usa raise() per spostarlo sopra al gruppo corrente

      tooltip.style("display", "block")
        .html(`<strong>${d.properties.name}</strong><br>✈ Flights: ${d.properties.flights}`);
    })
    .on("mousemove", event => {
      tooltip.style("top", `${event.pageY + 10}px`)
        .style("left", `${event.pageX + 10}px`);
    })
    .on("mouseout", function (event, d) {
      d3.select(this)
        .attr("fill", d => colorScale(d.properties.flights)) // Reset to original color
        .attr("stroke", "#222") // Reset stroke color
        .transition().duration(200) // Smooth transition
        .attr("transform", "translate(0,0) scale(1)"); // Ritorna alla dimensione originale

      tooltip.style("display", "none");
    });

  const quantiles = Array.isArray(colorScale.quantiles()) ? colorScale.quantiles() : [];

  // Ensure stateCounts is an array before using Math.min() & Math.max()
  const minCount = stateCounts.length > 0 ? Math.min(...stateCounts) : 0;
  const maxCount = stateCounts.length > 0 ? Math.max(...stateCounts) : 0;

  // Create bin ranges from quantiles
  const legendRanges = [minCount, ...quantiles.map(d => Math.round(d)), maxCount];

  if (legendRanges[legendRanges.length - 1] < maxCount) {
    legendRanges.push(maxCount);
  }

  // Generate range labels ensuring "+1 condition"
  const legendLabels = legendRanges.slice(0, -1).map((d, i) => {
    let nextValue = legendRanges[i + 1] - 1;
    if (i === legendRanges.length - 2) return `${d} -`; // Ensure last bin fully includes maxCount
    return `${d}-${nextValue}`;
  });

  const legend = svg.append("g")
    .attr("transform", `translate(-30, 0)`);

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

updateMap();
```

<br>
<div style="font-family: 'Times New Roman', serif;">
  <div style="display: flex; justify-content: center; align-items: center;">
    <div style="display: inline-block; width: 200px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color: #292929; color: white;">${selectedYear}</div>
    <div style="display: inline-block; width: 300px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color: #292929; color: white;">${selectedAirline1}</div>
  </div>
  <div class="grid grid-cols-1"> 
    <div class="card" style="display: flex; justify-content: center; align-items: center; height: 500px;"> 
      <div id="map-container"></div> 
    </div> 
  </div>
</div>

<div> 
The second visualization shifts focus from time to geography. This U.S. map shades each state based on the number of incoming flights, with deeper colors representing higher volumes. It immediately highlights which regions serve as major hubs—states like California, Texas, and Florida tend to show darker shades, reflecting their high connectivity and central role in domestic travel.

What's compelling here is the temporal filter. By selecting different years or filtering by airline, users can uncover how regional flight activity changed—perhaps observing a temporary dip in traffic to tourist-heavy states in 2020, followed by rebounds in 2021 and beyond. Note that the scale adjusts dynamically when filtering. This map translates abstract numbers into a spatial context, helping users grasp the real-world geography behind the data.

</div><br> 

 
 ## Monthly Flight Volume

 ```js
// Extract unique years and properly format them
const years = [...new Set(datasetFlights.map(d => new Date(d.FL_DATE).getFullYear()))]
  .sort((a, b) => a - b)
  .map(year => year.toString()); // Ensure it's treated as a string without commas

// Dropdown for year selection
const selYear = Inputs.select(years, { label: "📅 Select Year" });
const selAutoplay = document.createElement('button');
selAutoplay.id = "autoplay-btn";
selAutoplay.innerHTML = '▶ Play';
selAutoplay.style.background = "none";
selAutoplay.style.border = "none";
selAutoplay.style.padding = "0px 20px";
selAutoplay.style.cursor = "pointer";
document.querySelector("#autoplay-container").appendChild(selAutoplay);

// Variabili per il controllo dell'autoplay
let autoplayInterval;
let autoplayActive = false;

function toggleAutoplay() {
  const autoplayButton = document.getElementById("autoplay-btn");
  
  if (autoplayActive) {
    clearInterval(autoplayInterval);
    autoplayActive = false;
    autoplayButton.innerHTML = '▶  Play';
  } else {
    autoplayActive = true;
    autoplayButton.innerHTML = '■  Stop';
    
    function autoplayStep() {
      let currentIndex = years.indexOf(selYear.value);
      
      if (currentIndex === years.length - 1) { 
        setTimeout(() => {
          selYear.value = years[0]; // Torna al primo anno
          updateChart();
          autoplayInterval = setInterval(autoplayStep, 1500);
        }, 2500); 
        
        clearInterval(autoplayInterval); // Ferma temporaneamente il loop
      } else {
        currentIndex = (currentIndex + 1) % years.length; 
        selYear.value = years[currentIndex];
        updateChart();
      }
    }
    function updateChart() {
      document.querySelector("#ridgeline-container").innerHTML = "";
      document.querySelector("#ridgeline-container").appendChild(radarChart(selYear.value));
    }
    autoplayInterval = setInterval(autoplayStep, 1500);
  }
}
selAutoplay.addEventListener("click", toggleAutoplay);


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
function radarChart(year, { width = 500, height = 500 } = {}) {
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
selYear.addEventListener("input", () => {
  document.querySelector("#ridgeline-container").innerHTML = "";
  document.querySelector("#ridgeline-container").appendChild(radarChart(selYear.value));
});

```
<br>
<div style="font-family: 'Times New Roman', serif;">
  <div style="display: flex; justify-content: center; align-items: center;">
    <div style="display: inline-block; width: 200px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color: #292929; color: white;">${selYear}</div>
    <div style="display: inline-block; width: 85px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color: #292929; color: white;"><div id="autoplay-container"></div></div>
  </div>
  <div class="grid grid-cols-1"> <div class="card" style="display: flex; justify-content: center; align-items: center; height: 500px; " id="ridgeline-container"> ${ridgelineChart} </div> </div>
</div>


<div>
Lastly, the radar chart offers a fresh perspective by displaying monthly flight volumes in a circular, clock-like form. Each "spoke" of the chart represents a month, with the distance from the center indicating flight volume. This design cleverly visualizes seasonality in air travel—highlighting, for instance, the summer peaks or the holiday spikes in November and December.

What’s notable here is the ability to switch between years. This allows viewers to see how certain seasonal patterns persist or shift over time, and how anomalies—like the spring 2020 collapse—stand out starkly. For instance, a comparison between 2019 and 2020 reveals how travel during typically busy months was severely disrupted.

</div>