---
theme: dark
title: Flight Delays
toc: true
---


# Flight Delays ⏳

<br>

<div>
Analyzing flight delays offers valuable insights into the operational efficiency of airlines and the overall health of the aviation industry. The "Flight Delays" dashboard provides an interactive platform to explore various dimensions of flight delays across U.S. airlines and airports. Through a series of visualizations, users can uncover patterns related to timing, causes, distances, and geographic distribution of delays. Each graph is designed with interactive features that allow for a deeper understanding of the data.
</div>

<br>
<br>

## Flight Delays by Day & Hour 

<br>

```js
const datasetFlights = await FileAttachment("data/flights_data.csv").csv({ typed: true });
const cordinates = await FileAttachment("data/cordinates.csv").csv({ typed: true });

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

function getGlobalDelayRange(month) {
  const allYears = [...new Set(datasetFlights.map(d => new Date(d.FL_DATE).getFullYear()))];

  let globalMinAvgDelay = Infinity;
  let globalMaxAvgDelay = -Infinity;

  allYears.forEach(year => {
    const allDataForYear = datasetFlights.filter(d => {
      const flightMonth = new Date(d.FL_DATE).getMonth();
      return (month === "All" || flightMonth === availableMonths.indexOf(month)-1) && new Date(d.FL_DATE).getFullYear() === year;
    });

    const computedAverages = processHeatmapData(allDataForYear);
    const yearMinAvgDelay = d3.min(computedAverages, d => d.avgDelay);
    const yearMaxAvgDelay = d3.max(computedAverages, d => d.avgDelay);

    globalMinAvgDelay = Math.min(globalMinAvgDelay, yearMinAvgDelay);
    globalMaxAvgDelay = Math.max(globalMaxAvgDelay, yearMaxAvgDelay);
  });

  return {
    min: globalMinAvgDelay,
    max: globalMaxAvgDelay
  };
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

function hasDataForMonth(year, month) {
  const filteredData = filterData(year, month);
  return filteredData.length > 0;
}

function filterData(year, month) {
  const filtered = datasetFlights.filter(d => {
    const flightDate = new Date(d.FL_DATE);
    const flightYear = flightDate.getFullYear();
    const flightMonth = flightDate.getMonth();
    return flightYear === Number(year) && (month === "All" || flightMonth === availableMonths.indexOf(month) - 1);
  });
  return filtered;
}

function showNoDataMessage() {
  const container = d3.select("#heatmap-container");
  container.html("");
  container.append("div")
    .attr("class", "no-data-message")
    .style("text-align", "center")
    .style("color", "red")
    .style("font-size", "20px")
    .text("No data for this month");
}

function drawHeatmap() {
  d3.select(".no-data-message").remove();
  d3.select(".tooltip").remove();

  const filteredData = filterData(selectedYear.value, selectedMonth.value);

  if (filteredData.length === 0) {
    showNoDataMessage();
    return;
  }

  const heatmapData = processHeatmapData(filteredData);

  let globalMinAvgDelay, globalMaxAvgDelay;

  function updateGlobalLegend(month) {
    const range = getGlobalDelayRange(month);
    globalMinAvgDelay = range.min;
    globalMaxAvgDelay = range.max;
  }

  updateGlobalLegend(selectedMonth.value);

  const width = 900, height = 450, margin = { top: 120, right: 50, bottom: 50, left: 50 };

  const xScale = d3.scaleBand().domain(d3.range(0, 24)).range([margin.left, width - margin.right]).padding(0.05);
  const yScale = d3.scaleBand().domain(["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"]).range([margin.top, height - margin.bottom]).padding(0.05);
  
  const colorScale = d3.scaleDiverging()
    .domain([globalMaxAvgDelay, 0, globalMinAvgDelay])
    .interpolator(d3.interpolateRdYlGn);

  selectedMonth.addEventListener("input", () => {
    updateGlobalLegend(selectedMonth.value);
    drawHeatmap();
  });

  const container = d3.select("#heatmap-container");
  const legendExists = !container.select(".legend-group").empty();
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
  if (!legendExists) {
    const legendWidth = 300, legendHeight = 15;
    const legendX = (width - legendWidth) / 2;
    const legendSvg = svg.append("g").attr("transform", `translate(${legendX}, ${margin.top - 90})`);

    const legendScale = d3.scaleLinear()
      .domain([globalMinAvgDelay, 0, globalMaxAvgDelay]) // 0 è centrato
      .range([0, legendWidth / 2, legendWidth]);

    const legendAxis = d3.axisBottom(legendScale)
      .tickValues([globalMinAvgDelay, globalMaxAvgDelay, 0])
      .tickFormat(d => { return d === 0 ? "" : `${Math.round(d)} min`;} )
      .tickSizeOuter(0);

    const legendGradient = legendSvg.append("defs").append("linearGradient")
      .attr("id", "legend-gradient")
      .attr("x1", "0%").attr("y1", "0%").attr("x2", "100%").attr("y2", "0%");

    legendGradient.append("stop").attr("offset", "0%").attr("stop-color", colorScale(globalMinAvgDelay));
    legendGradient.append("stop").attr("offset", "50%").attr("stop-color", colorScale(0));
    legendGradient.append("stop").attr("offset", "100%").attr("stop-color", colorScale(globalMaxAvgDelay));

    legendSvg.append("rect").attr("width", legendWidth).attr("height", legendHeight).style("fill", "url(#legend-gradient)");

    const legendAxisGroup = legendSvg.append("g").attr("transform", `translate(0, ${legendHeight})`).call(legendAxis);

    // Append a white line for zero
    legendAxisGroup.selectAll(".tick")
      .filter(d => d === 0)
      .append("line")
      .attr("y1", -legendHeight)
      .attr("y2", 0)
      .attr("stroke", "black")
      .attr("stroke-width", 2);

    legendSvg.append("text").attr("x", legendWidth / 2).attr("y", -10).attr("text-anchor", "middle").style("fill", "white").text("Avg Delay (min)");
  }
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

    function autoplayStep() {
      d3.select(".tooltip").remove();

      let currentYearIndex = availableYears.indexOf(selectedYear.value);
      let nextYear = availableYears[currentYearIndex + 1];

      let nextMonth = selectedMonth.value;

      while (!hasDataForMonth(nextYear, nextMonth) && nextYear !== availableYears[availableYears.length - 1]) {
        nextYear = availableYears[availableYears.indexOf(nextYear) + 1];
      }

      if (!hasDataForMonth(nextYear, nextMonth)) {
        nextYear = availableYears[0]; 
      }

      if (hasDataForMonth(nextYear, nextMonth)) {
        selectedYear.value = nextYear;
        selectedMonth.value = nextMonth;
        drawHeatmap();
      } else {
        selectedYear.value = availableYears[0];
        selectedMonth.value = "All";
        drawHeatmap();
      }
    }

    autoplayInterval = setInterval(autoplayStep, 1500);
  }
}


document.getElementById("autoplay-btn").addEventListener("click", toggleAutoplay);

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

<div>
This heatmap visualizes the average delay times for U.S. airlines throughout 2022, segmented by days of the week and hours of the day. The y-axis lists the days from Sunday to Saturday, while the x-axis represents the 24-hour clock. Color intensity conveys the average delay duration:

- Green Shades: Indicate shorter delays or early arrivals.
- White Line: Represents zero delay, marking on-time departures.
- Red Shades: Denote longer delays.
</div>
<div>
Analyzing delays based on time of day allows users to easily identify peak congestion periods. By observing the color concentrations across different hours, it becomes clear when delays are most frequent, helping to pinpoint patterns such as morning rushes or late-night bottlenecks. <br><br>
Similarly, examining trends across different days of the week can reveal broader patterns in flight delays. Some days may consistently experience higher disruptions due to increased air traffic, maintenance schedules, or external factors like weather conditions. Identifying these trends can provide valuable insights for both travellers and airline operators, helping to anticipate and mitigate potential delays.<br><br>
By interacting with the heatmap, users can pinpoint specific times and days that are more prone to delays, aiding in strategic planning for travel or operational adjustments.

</div>
<br>

## Number or Percentage of Delays by Reason

<br>



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
const viewToggle = Inputs.radio(["Percentage", "Absolute Numbers"], {
  label: "📊 View Mode",
  value: "Percentage" // Default mode
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
      const count = d.data[season] || 0;

      d3.select(this).style("opacity", 0.7);
      tooltip.style("display", "block")
        .html(`<strong>${d.data.category}</strong><br>Season: ${season}<br>${percentageView ? `Percentage: ${percentage}% <br> Flights: ${count}` : `Delays: ${percentage}`}
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
This bar chart breaks down the causes of flight delays into five main categories:

1. Carrier: Issues directly related to the airline's operations.
2. NAS (National Aviation System): Delays due to the broader air traffic control system.
3. Late Aircraft: Delays caused by the late arrival of the incoming aircraft.
4. Weather: Weather-related disruptions.
5. Security: Delays stemming from security concerns or procedures.
</div>
<div>
Each bar is segmented by season (Winter, Spring, Summer, Fall), allowing users to observe how delay reasons fluctuate throughout the year.
Users can switch between viewing absolute numbers of delays and percentage distributions, providing both a macro and micro perspective on the data.<br><br>
This visualization enables users to discern which factors predominantly contribute to delays and how their impact varies seasonally, facilitating targeted strategies to mitigate specific delay causes.
</div>


<br>
<br>


## Flight distance vs. average delay per airline 

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
const scatterMargin = { top: 50, right: 50, bottom: 80, left: 50 };

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

<div>
The bubble chart examines the relationship between average flight distance and average delay time across different airlines. The x-axis represents the average flight distance in miles, while the y-axis shows the average delay in minutes. Each bubble corresponds to an airline, with:

- Bubble Size: Indicating the number of flights considered in the average.
- Bubble Position: Revealing the correlation between flight distance and delay duration for that airline.
</div>
<div>
Users can hover over each bubble to view specific data points, including the airline's name, average flight distance, average delay, and total number of flights.<br><br>
This chart assists in understanding whether longer flights are more susceptible to delays and how each airline's performance compares in this context, offering insights into operational efficiencies related to flight length.</div>
<br>
<br>


## Flight Delays for company 

```js
// Load necessary D3 libraries and US States GeoJSON
const topojson = await import("https://cdn.jsdelivr.net/npm/topojson@3/+esm");
const usMap = await d3.json("https://cdn.jsdelivr.net/npm/us-atlas@3/states-10m.json");

// Assign Seasons to Flights
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

// Compute Average Delay per Airport
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

// Create a mapping of airport codes to coordinates from the cordinates CSV
const airportCoords = cordinates.reduce((acc, d) => {
  acc[d.NAME] = [d.LONG, d.LAT];
  return acc;
}, {});

// Merge Delay Data with Coordinates
const airportData = airportDelays
  .filter(d => airportCoords[d.airport])
  .map(d => ({
    airport: d.airport,
    avgDelay: d.avgDelay,
    totalFlights: d.totalFlights,
    coords: airportCoords[d.airport]
  }));


// Set Map Dimensions
const width = 800, height = 550;

// Projection & Path Generator
const projection = d3.geoAlbersUsa().fitSize([width, height], topojson.feature(usMap, usMap.objects.states));
const path = d3.geoPath().projection(projection);

// Select Container & Remove Old SVG
const container = d3.select("#airport-map-container");
container.select("svg").remove();

// Create SVG
const svg = container.append("svg")
  .attr("width", width)
  .attr("height", height)
  .style("font", "12px sans-serif");

// Tooltip
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

// Draw US States
svg.append("g")
  .selectAll("path")
  .data(topojson.feature(usMap, usMap.objects.states).features)
  .join("path")
  .attr("d", path)
  .attr("fill", "#2c3e50") // Dark background for map
  .attr("stroke", "#fff");

// Define Color Scale for Delays
const colorScale = d3.scaleDiverging()
  .domain([30, 0, -10]) // Negative = Early, 0 = On Time, 30+ = Very Late
  .interpolator(d3.interpolateRdYlGn); // Red = Late, White = On-time, Green = Early

// Define Size Scale for Flights
const sizeScale = d3.scaleSqrt()
  .domain([0, d3.max(airportData, d => d.totalFlights)])
  .range([5, 30]); // Circle size

  airportData.forEach(d => {
    if (!projection(d.coords)) console.warn("Invalid projection:", d.airport, d.coords);
  });

// Draw Airport Circles
svg.append("g")
  .selectAll("circle")
  .data(airportData)
  .join("circle")
  .attr("cx", d => d.coords ? projection(d.coords)[0] : 0) 
  .attr("cy", d => d.coords ? projection(d.coords)[1] : 0)
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

// Create Season Toggle
const selectedSeason = Inputs.radio(["All", "Winter", "Spring", "Summer", "Fall"], {
  label: "🌍 Select Season",
  value: "All"
});

// Function to Update Map Based on Season
function updateMap() {
  const filteredData = selectedSeason.value === "All"? airportData
    : airportData.filter(d => datasetFlights.some(f => f.ORIGIN === d.airport && f.SEASON === selectedSeason.value));

  svg.selectAll("circle")
    .data(filteredData, d => d.airport)
    .join(
      enter => enter.append("circle")
        .attr("cx", d => d.coords ? projection(d.coords)[0] : 0) 
        .attr("cy", d => d.coords ? projection(d.coords)[1] : 0)
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

  // Define the color scale legend (Update the legend as well)
  const legend = svg.append("g")
    .attr("transform", `translate(590 20)`); // Position the legend on the right

  // Add a gradient for the color scale
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

  // Add a rectangle to display the color gradient
  legend.append("rect")
    .attr("width", 120)
    .attr("height", 20)
    .style("fill", "url(#color-legend)");

  // Add labels to explain the gradient
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

// Listen for Season Toggle Changes
selectedSeason.addEventListener("input", updateMap);

// Initial Map Render
updateMap();
```


<div class="grid grid-cols-1">
  <div class="card" style="display: flex; justify-content: center; align-items: center;">
    <div id="airport-map-container"></div>
  </div>
</div>

<div>
This U.S. map displays average flight delays at various airports, with each circle representing a specific airport. The visualization uses color and size to convey information:

  - Red: Indicates airports where flights tend to be delayed.
  - Yellow/Orange: Suggests generally on-time performance.
  - Green: Denotes airports where flights often arrive early.
</div><div>
The circle size reflects the volume of flights at that airport, with larger circles representing higher traffic.
By hovering over a circle, users can access detailed statistics about that airport's average delay and flight volume.
This geographic visualization allows users to identify regional patterns in flight delays, highlighting airports that may require operational improvements or those that excel in maintaining schedules.
</div>
