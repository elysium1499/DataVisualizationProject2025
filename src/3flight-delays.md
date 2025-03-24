---
theme: dashboard
title: Flight Delays
toc: true
---


# Flight Delays ⏳

<br>


## Flight Delays by Day & Hour (Heatmap)

<br>

```js
const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });
```

```js
// Load necessary D3 libraries
const d3 = await import("https://cdn.jsdelivr.net/npm/d3@7/+esm");

// Convert Time (DEP_TIME) to Hour Slots
function getHour(depTime) {
  if (!depTime) return null;
  const hour = Math.floor(depTime / 100);
  return hour < 24 ? hour : null;
}

// Convert Date to Day of the Week
function getDayOfWeek(date) {
  const days = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
  return days[new Date(date).getDay()];
}

// Extract Available Years and Months
const availableYears = [...new Set(datasetFlights.map(d => new Date(d.FL_DATE).getFullYear()))]
  .sort((a, b) => a - b)
  .map(String);

const availableMonths = ["All", ...Array.from({ length: 12 }, (_, i) => 
  new Date(2000, i, 1).toLocaleString("en-US", { month: "long" })
)];

const selectedYear = Inputs.select(availableYears, { label: "📆 Select Year" });
const selectedMonth = Inputs.select(availableMonths, { label: "📅 Select Month" });

let autoplayInterval;
let isAutoplayActive = false;

function filterData(year, month) {
  return datasetFlights.filter(d => {
    const flightDate = new Date(d.FL_DATE);
    const flightYear = flightDate.getFullYear();
    const flightMonth = flightDate.getMonth();
    
    return flightYear === Number(year) && (month === "All" || flightMonth === availableMonths.indexOf(month) - 1);
  });
}


function processHeatmapData(data) {
  const allDays = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
  const allHours = d3.range(0, 24);
  const heatmapMatrix = [];
  
  allDays.forEach(day => {
    allHours.forEach(hour => {
      heatmapMatrix.push({ day, hour, avgDelay: 0 });
    });
  });

  const computedData = d3.rollups(
    data.map(d => ({
      day: getDayOfWeek(d.FL_DATE),
      hour: getHour(d.DEP_TIME),
      delay: +d.ARR_DELAY
    })).filter(d => d.hour !== null),
    v => d3.mean(v, d => d.delay),
    d => d.day,
    d => d.hour
  ).map(([day, hours]) => 
    hours.map(([hour, avgDelay]) => ({
      day, hour, avgDelay: avgDelay || 0
    }))
  ).flat();

  computedData.forEach(d => {
    const index = heatmapMatrix.findIndex(h => h.day === d.day && h.hour === d.hour);
    if (index !== -1) {
      heatmapMatrix[index].avgDelay = d.avgDelay;
    }
  });

  return heatmapMatrix;
}

function drawHeatmap() {
  d3.select(".tooltip").remove();

  const filteredData = filterData(selectedYear.value, selectedMonth.value);
  const heatmapData = processHeatmapData(filteredData);

  const width = 900, height = 500, margin = { top: 120, right: 20, bottom: 50, left: 100 };

  const xScale = d3.scaleBand().domain(d3.range(0, 24)).range([margin.left, width - margin.right]).padding(0.05);
  const yScale = d3.scaleBand().domain(["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"]).range([margin.top, height - margin.bottom]).padding(0.05);

  const minDelay = d3.min(heatmapData, d => d.avgDelay);
  const maxDelay = d3.max(heatmapData, d => d.avgDelay);
  const colorScale = d3.scaleDiverging().domain([minDelay, 0, maxDelay]).interpolator(d3.interpolateRdYlGn);

  const container = d3.select("#heatmap-container");
  container.select("svg").remove();

  const svg = container.append("svg")
    .attr("width", width)
    .attr("height", height);

  // Create the tooltip
  const tooltip = d3.select("body").append("div")
    .attr("class", "tooltip")
    .style("position", "absolute")
    .style("visibility", "hidden")
    .style("background-color", "rgba(0,0,0,0.7)")
    .style("color", "#fff")
    .style("padding", "8px")
    .style("border-radius", "4px")
    .style("pointer-events", "none");

  svg.append("g")
    .selectAll("rect")
    .data(heatmapData)
    .join("rect")
    .attr("x", d => xScale(d.hour))
    .attr("y", d => yScale(d.day))
    .attr("width", xScale.bandwidth())
    .attr("height", yScale.bandwidth())
    .attr("fill", d => colorScale(d.avgDelay))
    .attr("stroke", "#222")
    .on("mouseover", function(event, d) {
      tooltip.style("visibility", "visible")
        .text(`Avg Delay: ${d.avgDelay.toFixed(2)} min`)
        .style("left", `${event.pageX + 10}px`)
        .style("top", `${event.pageY - 28}px`);
    })
    .on("mousemove", function(event) {
      tooltip.style("left", `${event.pageX + 10}px`)
        .style("top", `${event.pageY - 28}px`);
    })
    .on("mouseout", function() {
      tooltip.style("visibility", "hidden");
    });

  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(xScale).tickFormat(d => `${d}:00`));

  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(yScale));

  // Centered Legend
  const legendWidth = 600, legendHeight = 15;
  const legendX = (width - legendWidth) / 2;
  const legendSvg = svg.append("g").attr("transform", `translate(${legendX}, ${margin.top - 90})`);

  const legendScale = d3.scaleLinear().domain([minDelay, maxDelay]).range([0, legendWidth]);
  const legendAxis = d3.axisBottom(legendScale)
    .tickValues([minDelay, Math.round(maxDelay / 2), maxDelay, 0]) // Add 0 to the tick values
    .tickFormat(d => {
      return d === 0 ? "" : `${Math.round(d)} min`; // Hide text for 0
    })
    .tickSizeOuter(0); // Remove the outer ticks

  const legendGradient = legendSvg.append("defs").append("linearGradient")
    .attr("id", "legend-gradient")
    .attr("x1", "0%").attr("y1", "0%").attr("x2", "100%").attr("y2", "0%");

  legendGradient.append("stop").attr("offset", "0%").attr("stop-color", colorScale(maxDelay));
  legendGradient.append("stop").attr("offset", "50%").attr("stop-color", colorScale(0));
  legendGradient.append("stop").attr("offset", "100%").attr("stop-color", colorScale(minDelay));

  legendSvg.append("rect").attr("width", legendWidth).attr("height", legendHeight).style("fill", "url(#legend-gradient)");

  const legendAxisGroup = legendSvg.append("g").attr("transform", `translate(0, ${legendHeight})`).call(legendAxis);

  // Append a white line for zero
  legendAxisGroup.selectAll(".tick")
    .filter(d => d === 0)
    .append("line")
    .attr("y1", -legendHeight)
    .attr("y2", 0)
    .attr("stroke", "white")
    .attr("stroke-width", 2);

  legendSvg.append("text").attr("x", legendWidth / 2).attr("y", -10).attr("text-anchor", "middle").style("fill", "white").text("Avg Delay (min)");
}

function toggleAutoplay() {
  const autoplayButton = document.getElementById("autoplay-btn");
  
  if (isAutoplayActive) {
    clearInterval(autoplayInterval);
    isAutoplayActive = false;
    autoplayButton.innerHTML = '▶ Play';
  } else {
    isAutoplayActive = true;
    autoplayButton.innerHTML = '■ Stop';
    autoplayInterval = setInterval(() => {
      d3.select(".tooltip").remove();

      let currentYearIndex = availableYears.indexOf(selectedYear.value);
      if (currentYearIndex < availableYears.length - 1) {
        selectedYear.value = availableYears[currentYearIndex + 1];
      } else {
        selectedYear.value = availableYears[0];
      }
      drawHeatmap();
    }, 1000); // change year every second (1000ms)
  }
}

// Event listener for the autoplay button
document.getElementById("autoplay-btn").addEventListener("click", toggleAutoplay);

// Initial call to draw the heatmap
drawHeatmap();
selectedYear.addEventListener("input", drawHeatmap);
selectedMonth.addEventListener("input", drawHeatmap);

```

<div style="font-family: 'Times New Roman', serif;">
  <div style="display: flex; justify-content: center; align-items: center;">
    <div style="display: inline-block; width: 300px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color:#292929; color: white;">${selectedYear}</div>
    <div style="display: inline-block; width: 300px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color:#292929; color: white;">${selectedMonth}</div>
    <div style="display: inline-block; width: 57px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color:#292929; color: white;"><button id="autoplay-btn">▶ Play</button></div>
  </div>
  <div class="grid grid-cols-1"><div class="card" style="display: flex; justify-content: center; align-items: center;"><div id="heatmap-container"></div></div>
</div>
</div>

<div>This heat map shows the average delay in minutes for US airlines in 2022. The y-axis represents days of the week (Sun-Sat) and the x-axis represents hours of the day (0:00 to 23:00). The intensity of the color indicates the average delay, with green representing shorter or even negative delays (early arrivals), the white line representing no delay or advance (0 min), and red representing longer delays.</div>
<br>

## Number or Percentage of Delays by Reason (Bar chart)

<div>

MCO	Orlando Intl	Florida (FL)	High leisure travel volume

SEA	Seattle-Tacoma Intl	Washington (WA)	Important West Coast gateway

</div>

```js
//  Define Seasons
const seasons = ["Winter", "Spring", "Summer", "Fall"];

//  Function to Assign Season to Each Flight
function getSeason(date) {
  const month = new Date(date).getMonth() + 1; // Convert to 1-based month
  if ([12, 1, 2].includes(month)) return "Winter";
  if ([3, 4, 5].includes(month)) return "Spring";
  if ([6, 7, 8].includes(month)) return "Summer";
  return "Fall"; // September, October, November
}

//  Assign Season to Each Flight (Preprocessed for Speed)
datasetFlights.forEach(d => {
  d.FL_DATE = new Date(d.FL_DATE);
  d.SEASON = getSeason(d.FL_DATE);
});

//  Define Delay Categories
const delayCategories = ["Carrier", "NAS", "Late Aircraft", "Weather", "Security"];

//  Compute Delay Counts per (Delay Type, Season)
const delayCounts = d3.rollups(
  datasetFlights.filter(d => delayCategories.includes(d.DELAY_CATEGORY)), // Only relevant delays
  v => v.length, // Count flights per delay type
  d => d.DELAY_CATEGORY,
  d => d.SEASON
);

//  Convert Data to Percentage Format
function getStackedData(percentageView) {
  const rawData = Object.fromEntries(delayCounts.map(([category, seasons]) => [
    category,
    Object.fromEntries(seasons)
  ]));

  const stackedData = delayCategories.map(category => {
    const categoryData = rawData[category] || {};
    const totalDelays = d3.sum(Object.values(categoryData));

    return {
      category,
      ...categoryData,
      total: percentageView ? totalDelays : 1, // Normalize for percentage view
    };
  });

  return stackedData;
}

//  Create View Toggle (Absolute vs Percentage)
const viewToggle = Inputs.radio(["Absolute Numbers", "Percentage"], {
  label: "📊 View Mode",
  value: "Absolute Numbers" // Default mode
});

//  Create Reset Zoom Button
const resetZoomButton = Inputs.button("🔍 Reset Zoom");

//  Define Chart Dimensions
const width = 900, height = 500, margin = { top: 50, right: 80, bottom: 80, left: 100 };

//  Track Selected Category for Zoom
let selectedCategory = null;

//  Create Stacked Chart
function drawStackedBarChart() {
  const percentageView = viewToggle.value === "Percentage";
  let data = getStackedData(percentageView);

  //  Filter data if zoomed on a specific category
  if (selectedCategory) {
    data = data.filter(d => d.category === selectedCategory);
  }

  //  Get Dynamic Y-Axis Max Value
  const maxTotal = d3.max(data, d => d3.sum(seasons.map(season => d[season] || 0)));

  //  Define Scales
  const xScale = d3.scaleBand()
    .domain(data.map(d => d.category))
    .range([margin.left, width - margin.right])
    .padding(0.3);

  const yScaleLeft = d3.scaleLinear()
    .domain([0, percentageView ? 100 : maxTotal]) // Adjust for percentage view
    .nice()
    .range([height - margin.bottom, margin.top]);

  const yScaleRight = d3.scaleLinear()
    .domain([0, 100]) // Percentage always 0-100%
    .nice()
    .range([height - margin.bottom, margin.top]);

  // Define the custom color scale
  const colorScale = d3.scaleOrdinal()
    .domain(seasons)
    .range(["#8cacc6", "#e598be", "#93c47d", "#fbb95d"]); // Light blue, light pink, light green, light orange

  //  Select Container
  const container = d3.select("#stacked-chart-container");

  //  Remove Old SVG
  container.select("svg").remove();

  //  Create SVG
  const svg = container.append("svg")
    .attr("width", width)
    .attr("height", height)
    .style("font", "12px sans-serif");

  //  Tooltip
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

  //  Stack Data
  const stackedSeries = d3.stack()
    .keys(seasons)
    .value((d, key) => (d[key] || 0) / d.total * (percentageView ? 100 : 1)) // Normalize for percentage
    (data);

  //  Draw Stacked Bars
  svg.append("g")
    .selectAll("g")
    .data(stackedSeries)
    .join("g")
    .attr("fill", d => colorScale(d.key))
    .selectAll("rect")
    .data(d => d)
    .join("rect")
    .attr("x", d => xScale(d.data.category))
    .attr("y", d => yScaleLeft(d[1]))
    .attr("height", d => yScaleLeft(d[0]) - yScaleLeft(d[1]))
    .attr("width", xScale.bandwidth())
    .on("mouseover", function(event, d) {
      const season = d3.select(this.parentNode).datum().key;
      const percentage = Math.round(d[1] - d[0]);

      d3.select(this).style("opacity", 0.7);
      tooltip.style("display", "block")
        .html(`
          <strong>${d.data.category}</strong><br>
          Season: ${season}<br>
          ${percentageView ? "Percentage: " : "Delays: "} ${percentage}${percentageView ? "%" : ""}
        `);
    })
    .on("mousemove", event => {
      tooltip.style("top", `${event.pageY + 10}px`).style("left", `${event.pageX + 10}px`);
    })
    .on("mouseout", function () {
      d3.select(this).style("opacity", 1);
      tooltip.style("display", "none");
    })
    .on("click", function(event, d) {
      selectedCategory = selectedCategory === d.data.category ? null : d.data.category;
      //  Hide tooltip when zooming
      tooltip.style("display", "none");
      drawStackedBarChart(); //  Redraw chart with zoom
    });

  //  X Axis (Delay Categories)
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(xScale))
    .selectAll("text")
    .style("fill", "white")
    .style("font-size", "14px");

  //  Y Axis (Left - Absolute/Percentage)
  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(yScaleLeft).tickFormat(d => percentageView ? `${d}%` : d))
    .selectAll("text")
    .style("fill", "white")
    .style("font-size", "14px");
  const legend = svg.append("g")
    .attr("transform", `translate(${width - margin.right + 20}, ${margin.top})`)
    .style("font-size", "12px");

  //  Legend
  seasons.forEach((season, i) => {
    const legendRow = legend.append("g")
      .attr("transform", `translate(-40, ${i * 20})`);

    legendRow.append("rect")
      .attr("width", 15)
      .attr("height", 15)
      .attr("fill", colorScale(season));

    legendRow.append("text")
      .attr("x", 20)
      .attr("y", 13)
      .attr("text-anchor", "start")
      .text(season)
      .style("fill", "white");
  });

  //  Reset Zoom Button
  resetZoomButton.onclick = () => {
    selectedCategory = null;
    drawStackedBarChart();
  };
}

//  Initial Render
drawStackedBarChart();

//  Update Chart when View Changes
viewToggle.addEventListener("input", drawStackedBarChart);
```
<div style="font-family: 'Times New Roman', serif;">
  <div style="display: flex; justify-content: center; align-items: center;">
    <div style="display: inline-block; width: 110px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color:#292929; color: white;">${resetZoomButton}</div>
    <div style="display: inline-block; width: 380px; padding: 8px 5px; border: 1px solid; border-radius: 8px; margin-right: 10px; background-color:#292929; color: white;"><div class="filter"> ${viewToggle} </div></div>
  </div>
  <div class="grid grid-cols-1"><div class="card" style="display: flex; justify-content: center; align-items: center;"><div id="stacked-chart-container"></div></div></div>
</div>

<div>
The x-axis lists the major delay categories: Carrier, NAS (National Aviation System), Late Aircraft, Weather, and Security. The y-axis represents the absolute number of delays.
Each bar is divided into colored segments, and the height of each segment indicates the number of delays attributed to a specific factor within that major category. The different colored sections within the bars show the number of those delays divided by season (Winter, Spring, Summer, Fall). In addition, it is possible to view the percentage distribution of flight delays, on the y-axis, for different causes, by changing the view using the button.
</div>


<br>


## Flight distance vs. average delay per airline (Bubble chart)

```js
//  Make a deep copy of dataset to avoid mutations
const scatterData = JSON.parse(JSON.stringify(datasetFlights));

//  Process Data for Scatter Plot
const processedScatterData = scatterData.map(d => ({
  airline: d.AIRLINE,
  distance: +d.DISTANCE, // Convert to number
  delay: +d.ARR_DELAY    // Convert to number
})).filter(d => !isNaN(d.distance) && !isNaN(d.delay)); //  Remove NaN values

//  Compute average delay & distance per airline
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

//  Step 1: Set up chart dimensions
const scatterWidth = 750, scatterHeight = 450;
const scatterMargin = { top: 50, right: 50, bottom: 80, left: 80 };

//  Step 2: Define Scales
const scatterXScale = d3.scaleLinear()
  .domain([0, d3.max(scatterStats, d => d.avgDistance) || 1000]) //  Default value to avoid undefined
  .range([scatterMargin.left, scatterWidth - scatterMargin.right]);

const scatterYScale = d3.scaleLinear()
  .domain([d3.min(scatterStats, d => d.avgDelay) - 1, d3.max(scatterStats, d => d.avgDelay) || 100]) //  Default value to avoid undefined
  .range([scatterHeight - scatterMargin.bottom, scatterMargin.top]);

const scatterSizeScale = d3.scaleSqrt()
  .domain([0, d3.max(scatterStats, d => d.numFlights) || 1000]) //  Default value to avoid undefined
  .range([5, 20]); // Point size based on number of flights

const scatterColorScale = d3.scaleOrdinal(d3.schemeTableau10)
  .domain(scatterStats.map(d => d.airline));

//  Step 3: Select the container div
const scatterContainer = d3.select("#scatter-container");

//  Remove old SVG to prevent duplication
scatterContainer.select("svg").remove();

//  Step 4: Create SVG
const scatterSvg = scatterContainer.append("svg")
  .attr("width", scatterWidth)
  .attr("height", scatterHeight)
  .style("font", "12px sans-serif");

//  Tooltip
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

//  Step 5: Draw Scatter Plot
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

//  Step 6: Add Axes
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

//  Add a dotted reference line at y = 0
scatterSvg.append("line")
  .attr("x1", scatterMargin.left)
  .attr("x2", scatterWidth - scatterMargin.right)
  .attr("y1", scatterYScale(0)) // Map 0 to the Y scale
  .attr("y2", scatterYScale(0))
  .attr("stroke", "white") // Line color (Change to preferred color)
  .attr("stroke-width", 1)
  .attr("stroke-dasharray", "5,5"); // Dotted line pattern
```

<div class="grid grid-cols-1">  
  <div class="card" style="display: flex; justify-content: center; align-items: center;">
    <div id="scatter-container"></div>
  </div>
</div> 

<div>This chart is a Bubble plot that focuses on data for airlines. It shows the relationship between the average flight distance (in miles) on the x-axis and the average delay (in minutes) on the y-axis.
The circle specifically represents an airline with several pieces of information about it: the average flight distance in miles, the average delay in minutes, the number of flights considered in the average.</div>
<br>


## Flight Delays for company (Map) 

```js
//  Load necessary D3 libraries and US States GeoJSON
const topojson = await import("https://cdn.jsdelivr.net/npm/topojson@3/+esm");
const usMap = await d3.json("https://cdn.jsdelivr.net/npm/us-atlas@3/states-10m.json");

//  Assign Seasons to Flights
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

//  Compute Average Delay per Airport
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

//  Load Airport Coordinates (Replace this with an actual airport dataset)
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


//  Merge Delay Data with Coordinates
const airportData = airportDelays
  .filter(d => airportCoords[d.airport])
  .map(d => ({
    airport: d.airport,
    avgDelay: d.avgDelay,
    totalFlights: d.totalFlights,
    coords: airportCoords[d.airport]
  }));

//  Set Map Dimensions
const width = 800, height = 550;

//  Projection & Path Generator
const projection = d3.geoAlbersUsa().fitSize([width, height], topojson.feature(usMap, usMap.objects.states));
const path = d3.geoPath().projection(projection);

//  Select Container & Remove Old SVG
const container = d3.select("#airport-map-container");
container.select("svg").remove();

//  Create SVG
const svg = container.append("svg")
  .attr("width", width)
  .attr("height", height)
  .style("font", "12px sans-serif");

//  Tooltip
const tooltip = d3.select("body").append("div")
  .attr("class", "tooltip")
  .style("position", "absolute")
  .style("background", "rgba(0, 0, 0, 0.9)")
  .style("color", "white")
  .style("padding", "6px 10px")
  .style("border-radius", "5px")
  .style("font-size", "12px")
  .style("pointer-events", "none")
  .style("display", "none")
  .style("text-align", "left"); // Aggiunta per allineare il testo a sinistra

//  Draw US States
svg.append("g")
  .selectAll("path")
  .data(topojson.feature(usMap, usMap.objects.states).features)
  .join("path")
  .attr("d", path)
  .attr("fill", "#2c3e50") // Dark background for map
  .attr("stroke", "#fff");

//  Define Color Scale for Delays
const colorScale = d3.scaleDiverging()
  .domain([30, 0, -10]) // Negative = Early, 0 = On Time, 30+ = Very Late
  .interpolator(d3.interpolateRdYlGn); // Red = Late, White = On-time, Green = Early

//  Define Size Scale for Flights
const sizeScale = d3.scaleSqrt()
  .domain([0, d3.max(airportData, d => d.totalFlights)])
  .range([5, 30]); // Circle size

//  Draw Airport Circles
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
  .style("pointer-events", "all") // Ensure circles are clickable
  .sort((a, b) => d3.descending(a.totalFlights, b.totalFlights)) // Ordina in modo decrescente in base al numero totale di voli
  .on("mouseover", function(event, d) {
    d3.select(this).attr("stroke", "white");
    tooltip.style("display", "block")
      .html(`
        <strong>${d.airport}</strong><br>
        ⏳ Avg Delay: ${Math.round(d.avgDelay)} mins<br>
        ✈ Flights: ${d.totalFlights}
      `);
  })
  .on("mousemove", event => {
    tooltip.style("top", `${event.pageY + 10}px`).style("left", `${event.pageX - (tooltip.node().offsetWidth / 2) - 10}px`);
  })
  .on("mouseout", function() {
    d3.select(this).attr("stroke", "#222");
    tooltip.style("display", "none");
  });

//  Create Season Toggle
const selectedSeason = Inputs.radio(["All", "Winter", "Spring", "Summer", "Fall"], {
  label: "🌍 Select Season",
  value: "All"
});

//  Function to Update Map Based on Season
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
        .attr("fill", d => {
          return colorScale(d.avgDelay); // Ensure avgDelay is correctly mapped to the color scale
        })
        .attr("stroke", "#222")
        .attr("opacity", 0.8)
        .on("mouseover", function (event, d) {
          d3.select(this).attr("stroke", "white");
          tooltip.style("display", "block").html(`
            <div style="text-align: left;">
              <strong>${d.airport}</strong><br>
              ⏳ Avg Delay: ${Math.round(d.avgDelay)} mins<br>
              ✈ Flights: ${d.totalFlights}
            </div>
          `);
        })
        .on("mousemove", event => {
          tooltip.style("top", `${event.pageY + 10}px`).style("left", `${event.pageX - (tooltip.node().offsetWidth / 2) - 10}px`);
        })
        .on("mouseout", function () {
          d3.select(this).attr("stroke", "#222");
          tooltip.style("display", "none");
        }),
      update => update.transition().duration(500)
        .attr("r", d => sizeScale(d.totalFlights))
        .attr("fill", d => colorScale(d.avgDelay)), // Use the correct color scale here
      exit => exit.remove()
    )
    .sort((a, b) => d3.descending(a.totalFlights, b.totalFlights)); // Ordina le bolle aggiornate

  //  Define the color scale legend (Update the legend as well)
  const legend = svg.append("g")
    .attr("transform", `translate(590 20)`); // Position the legend on the right

  //  Add a gradient for the color scale
  legend.append("defs")
    .append("linearGradient")
    .attr("id", "color-legend")
    .attr("x1", "0%")
    .attr("y1", "0%")
    .attr("x2", "100%")
    .attr("y2", "0%")
    .selectAll("stop")
    .data([
      { offset: "0%", color: colorScale(30) }, // Red (Late)
      { offset: "50%", color: colorScale(0) },  // White (On Time)
      { offset: "100%", color: colorScale(-10) } // Green (Early)
    ])
    .enter().append("stop")
    .attr("offset", d => d.offset)
    .attr("stop-color", d => d.color);

  //  Add a rectangle to display the color gradient
  legend.append("rect")
    .attr("width", 120)
    .attr("height", 20)
    .style("fill", "url(#color-legend)");

  //  Add labels to explain the gradient
  legend.append("text")
    .attr("x", 0)
    .attr("y", 40)
    .attr("text-anchor", "middle")
    .style("font-size", "12px")
    .style("fill", "white")
    .text("Late");

  legend.append("text")
    .attr("x", 60)
    .attr("y", 40)
    .attr("text-anchor", "middle")
    .style("font-size", "12px")
    .style("fill", "white")
    .text("On Time");

  legend.append("text")
    .attr("x", 120)
    .attr("y", 40)
    .attr("text-anchor", "middle")
    .style("font-size", "12px")
    .style("fill", "white")
    .text("Early");
}

//  Listen for Season Toggle Changes
selectedSeason.addEventListener("input", updateMap);

//  Initial Map Render
updateMap();
```


<div class="grid grid-cols-1">
  <div class="card" style="display: flex; justify-content: center; align-items: center;">
    <div id="airport-map-container"></div>
  </div>
</div>

<div>
This chart is a map of the United States showing average flight delays at different airports. Each circle on the map represents an airport.
The color of the circle indicates the average delay status: <br>
- Red suggests that flights at this airport tend to be delayed. <br>
- Yellow/orange suggests that flights are generally on time. <br>
- Green suggests that flights tend to be early. <br>
The size of the circle likely represents the volume of flights at that particular airport; larger circles would indicate a greater number of flights.
Hovering over or selecting an airport would provide more specific details about its average delay and number of flights.
</div>
