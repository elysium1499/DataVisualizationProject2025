---
theme: dashboard
title: Flight Delays ⏳
toc: true
---
 
# Flight Delays ⏳

<br>


## Flight Delays by Day & Hour


```js
// Load dataset
const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });

console.log("🚀 Interactive Heatmap Running!");

// ✅ Load necessary D3 libraries
const d3 = await import("https://cdn.jsdelivr.net/npm/d3@7/+esm");

// ✅ Convert Time (DEP_TIME) to Hour Slots
function getHour(depTime) {
  if (!depTime) return null;
  const hour = Math.floor(depTime / 100);
  return hour < 24 ? hour : null; // Ensure valid hours
}

// ✅ Convert Date to Day of the Week
function getDayOfWeek(date) {
  const days = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
  return days[new Date(date).getDay()];
}

// ✅ Extract Available Years and Months (Fixed Year Formatting)
const availableYears = [...new Set(datasetFlights.map(d => new Date(d.FL_DATE).getFullYear()))]
  .sort((a, b) => a - b) // Ensure sorting order
  .map(String); // Convert to string to avoid formatting issues

const availableMonths = ["All", ...Array.from({ length: 12 }, (_, i) => 
  new Date(2000, i, 1).toLocaleString("en-US", { month: "long" })
)];

// ✅ Create Dropdown Filters
const selectedYear = Inputs.select(availableYears, { label: "📆 Select Year" });
const selectedMonth = Inputs.select(availableMonths, { label: "📅 Select Month" });

// ✅ Function to Filter Dataset
function filterData(year, month) {
  return datasetFlights.filter(d => {
    const flightDate = new Date(d.FL_DATE);
    return flightDate.getFullYear() === Number(year) &&
      (month === "All" || flightDate.toLocaleString("en-US", { month: "long" }) === month);
  });
}

// ✅ Function to Process Data for Heatmap
function processHeatmapData(data) {
  // Initialize a full matrix for all (day, hour) combinations to prevent empty cells
  const allDays = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
  const allHours = d3.range(0, 24);
  const heatmapMatrix = [];

  allDays.forEach(day => {
    allHours.forEach(hour => {
      heatmapMatrix.push({ day, hour, avgDelay: 0 }); // Default 0 min delay
    });
  });

  // Compute average delays from filtered data
  const computedData = d3.rollups(
    data.map(d => ({
      day: getDayOfWeek(d.FL_DATE),
      hour: getHour(d.DEP_TIME),
      delay: +d.ARR_DELAY
    })).filter(d => d.hour !== null), // Remove invalid time entries
    v => d3.mean(v, d => d.delay), // Compute average delay per (day, hour)
    d => d.day,
    d => d.hour
  ).map(([day, hours]) => 
    hours.map(([hour, avgDelay]) => ({
      day, hour, avgDelay: avgDelay || 0 // Default to 0 if no data
    }))
  ).flat(); // Flatten nested structure

  // Merge computed data into initialized matrix
  computedData.forEach(d => {
    const index = heatmapMatrix.findIndex(h => h.day === d.day && h.hour === d.hour);
    if (index !== -1) {
      heatmapMatrix[index].avgDelay = d.avgDelay;
    }
  });

  return heatmapMatrix;
}

// ✅ Function to Draw Heatmap
function drawHeatmap() {
  console.log("📊 Selected Year:", selectedYear.value, "| Selected Month:", selectedMonth.value);

  const filteredData = filterData(selectedYear.value, selectedMonth.value);
  const heatmapData = processHeatmapData(filteredData);

  // ✅ Heatmap Dimensions
  const width = 900, height = 500, margin = { top: 100, right: 1, bottom: 50, left: 100 };

  // ✅ Define Scales
  const xScale = d3.scaleBand().domain(d3.range(0, 24)).range([margin.left, width - margin.right]).padding(0.05);
  const yScale = d3.scaleBand().domain(["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"]).range([margin.top, height - margin.bottom]).padding(0.05);

  // ✅ Define Updated Diverging Color Scale
  const minDelay = d3.min(heatmapData, d => d.avgDelay);
  const maxDelay = d3.max(heatmapData, d => d.avgDelay);
  const colorScale = d3.scaleDiverging().domain([maxDelay, 0, minDelay]).interpolator(d3.interpolateRdYlGn);

  // ✅ Select Heatmap Container
  const container = d3.select("#heatmap-container");
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

  // ✅ Draw Heatmap Squares
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
      d3.select(this).attr("stroke", "white");
      tooltip.style("display", "block")
        .html(`
          <strong>${d.day} at ${d.hour}:00</strong><br>
          ⏳ Avg Delay: ${Math.round(d.avgDelay)} mins
        `);
    })
    .on("mousemove", event => {
      tooltip.style("top", `${event.pageY + 10}px`).style("left", `${event.pageX + 10}px`);
    })
    .on("mouseout", function() {
      d3.select(this).attr("stroke", "#222");
      tooltip.style("display", "none");
    });

  // ✅ X Axis (Hours)
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(xScale).tickFormat(d => `${d}:00`))
    .selectAll("text")
    .style("fill", "white");

  // ✅ Y Axis (Days)
  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(yScale))
    .selectAll("text")
    .style("fill", "white");

  // ✅ Add Title
  //svg.append("text")
    //.attr("x", width / 2)
    //.attr("y", 20)
    //.attr("text-anchor", "middle")
    //.style("font-size", "16px")
    //.style("fill", "white")
    //.text("🔥 Heatmap of Flight Delays by Day & Hour");

  // ✅ ADD LEGEND
  const legendWidth = 250, legendHeight = 15;
  const legendSvg = svg.append("g").attr("transform", `translate(${width / 2}, ${margin.top - 70})`);

  const legendScale = d3.scaleLinear().domain([maxDelay, minDelay]).range([0, legendWidth]);
  const legendAxis = d3.axisBottom(legendScale).tickValues([maxDelay, minDelay]).tickFormat(d => `${Math.round(d)} min`);

  const legendGradient = legendSvg.append("defs").append("linearGradient")
    .attr("id", "legend-gradient")
    .attr("x1", "0%").attr("y1", "0%").attr("x2", "100%").attr("y2", "0%");

  legendGradient.append("stop").attr("offset", "0%").attr("stop-color", colorScale(maxDelay));
  legendGradient.append("stop").attr("offset", "50%").attr("stop-color", colorScale(0));
  legendGradient.append("stop").attr("offset", "100%").attr("stop-color", colorScale(minDelay));

  legendSvg.append("rect").attr("width", legendWidth).attr("height", legendHeight).style("fill", "url(#legend-gradient)");
  legendSvg.append("g").attr("transform", `translate(0, ${legendHeight})`).call(legendAxis).selectAll("text").style("fill", "white");

  legendSvg.append("text").attr("x", legendWidth / 2).attr("y", -10).attr("text-anchor", "middle").style("fill", "white").text("Avg Delay (min)");

}

// ✅ Run Heatmap Function Initially
drawHeatmap();

// ✅ Update on Filter Change
selectedYear.addEventListener("input", drawHeatmap);
selectedMonth.addEventListener("input", drawHeatmap);

```
<div style="display: flex; gap: 15px;">
  <div class="filter"> ${selectedYear} </div>
  <div class="filter"> ${selectedMonth} </div>
</div>
<div class="grid grid-cols-1">
  <div class="card">
    <div id="heatmap-container"></div>
  </div>
</div>


<br>

## Percentage of Delays by Reason 📊

```js
// Load dataset
const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });

console.log("🚀 Stacked Bar Chart Running!");

// ✅ Load necessary D3 libraries
const d3 = await import("https://cdn.jsdelivr.net/npm/d3@7/+esm");

// ✅ Define Seasons
const seasons = ["Winter", "Spring", "Summer", "Fall"];

// ✅ Function to Assign Season to Each Flight
function getSeason(date) {
  const month = new Date(date).getMonth() + 1; // Convert to 1-based month
  if ([12, 1, 2].includes(month)) return "Winter";
  if ([3, 4, 5].includes(month)) return "Spring";
  if ([6, 7, 8].includes(month)) return "Summer";
  return "Fall"; // September, October, November
}

// ✅ Assign Season to Each Flight (Preprocessed for Speed)
datasetFlights.forEach(d => {
  d.FL_DATE = new Date(d.FL_DATE);
  d.SEASON = getSeason(d.FL_DATE);
});

// ✅ Define Delay Categories
const delayCategories = ["Carrier", "NAS", "Late Aircraft", "Weather", "Security"];

// ✅ Compute Delay Counts per (Delay Type, Season)
const delayCounts = d3.rollups(
  datasetFlights.filter(d => delayCategories.includes(d.DELAY_CATEGORY)), // Only relevant delays
  v => v.length, // Count flights per delay type
  d => d.DELAY_CATEGORY,
  d => d.SEASON
);

// ✅ Convert Data to Percentage Format
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

// ✅ Create View Toggle (Absolute vs Percentage)
const viewToggle = Inputs.radio(["Absolute Numbers", "Percentage"], {
  label: "📊 View Mode",
  value: "Absolute Numbers" // Default mode
});

// ✅ Create Reset Zoom Button
const resetZoomButton = Inputs.button("🔍 Reset Zoom");

// ✅ Define Chart Dimensions
const width = 900, height = 500, margin = { top: 50, right: 80, bottom: 80, left: 100 };

// ✅ Track Selected Category for Zoom
let selectedCategory = null;

// ✅ Create Stacked Chart
function drawStackedBarChart() {
  console.log("📊 Drawing Chart in:", viewToggle.value, "| Zoomed on:", selectedCategory);

  const percentageView = viewToggle.value === "Percentage";
  let data = getStackedData(percentageView);

  // ✅ Filter data if zoomed on a specific category
  if (selectedCategory) {
    data = data.filter(d => d.category === selectedCategory);
  }

  // ✅ Get Dynamic Y-Axis Max Value
  const maxTotal = d3.max(data, d => d3.sum(seasons.map(season => d[season] || 0)));

  // ✅ Define Scales
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

  const colorScale = d3.scaleOrdinal(d3.schemeSet2)
    .domain(seasons);

  // ✅ Select Container
  const container = d3.select("#stacked-chart-container");

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

  // ✅ Stack Data
  const stackedSeries = d3.stack()
    .keys(seasons)
    .value((d, key) => (d[key] || 0) / d.total * (percentageView ? 100 : 1)) // Normalize for percentage
    (data);

  // ✅ Draw Stacked Bars
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
      // ✅ Hide tooltip when zooming
      tooltip.style("display", "none");
      drawStackedBarChart(); // ✅ Redraw chart with zoom
    });

  // ✅ X Axis (Delay Categories)
  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(xScale))
    .selectAll("text")
    .style("fill", "white")
    .style("font-size", "14px");

  // ✅ Y Axis (Left - Absolute/Percentage)
  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(yScaleLeft).tickFormat(d => percentageView ? `${d}%` : d))
    .selectAll("text")
    .style("fill", "white")
    .style("font-size", "14px");

  // ✅ Reset Zoom Button
  resetZoomButton.onclick = () => {
    selectedCategory = null;
    drawStackedBarChart();
  };
}

// ✅ Initial Render
drawStackedBarChart();

// ✅ Update Chart when View Changes
viewToggle.addEventListener("input", drawStackedBarChart);

```
<div class="grid grid-cols-1">
  <div class="card">
    <div id="stacked-chart-container"></div> <!-- Stacked bar chart container -->
  </div>
</div>

<div style="margin-bottom: 15px;">  ${resetZoomButton} </div>

<div style="display: flex; gap: 15px;">
  <div class="filter"> ${viewToggle} </div>
</div>

<p>


MCO	Orlando Intl	Florida (FL)	High leisure travel volume

SEA	Seattle-Tacoma Intl	Washington (WA)	Important West Coast gateway

</p>
