---
theme: dark
title: Flights Analysis
toc: true
---

<style>
:root {
  --theme-background: #121212;
  --theme-foreground: #ffffff;
  --theme-foreground-muted: #b0bec5;
  --theme-border: #444444;
  --theme-card-background:#2e2e2e;
}

body {
  background-color: var(--theme-background);
  color: var(--theme-foreground);
}

.navbar {
  display: flex;
  justify-content: center;
  gap: 0.2rem;
  padding: 10px;
  background: var(--theme-card-background);
  border-bottom: 1px solid var(--theme-border);
}

.navbar a {
  color: white;
  text-decoration: none;
  font-weight: bold;
}

.hero {
  margin: 2rem auto;
  display: flex; 
  flex-direction: column; 
  align-items: center; 
  text-align: center;"
}

.hero h1 {
  font-size: 2.5rem;
  color: #f39c12;
}

.hero h2 {
  font-size: 1.5rem;
  color: var(--theme-foreground-muted);
}

.cards-container {
  display: flex;
  gap: 1rem;
  justify-content: center;
  flex-wrap: wrap;
}

.card {
  padding: 1rem;
  border: 1px solid var(--theme-border);
  border-radius: 10px;
  background-color: var(--theme-card-background);
  text-align: center;
  width: 160px;
}

.card a {
  color: white;
  font-weight: bold;
}
</style>


<div class="hero">
  <h1>Flights Analysis</h1>
  <h2>Explore flight data and insights with our interactive dashboard.</h2>
</div>

## Welcome to Flights Analysis
<div>Have you ever wanted to better understand the world of aviation? Flights Analysis provides a comprehensive view of air travel, allowing you to explore detailed data on airlines, airports, delays, and global trends. Every day, thousands of flights cross the skies, connecting cities and nations.</div><br>

### 🚀 Why use this dashboard?
- 📊 Interactive Data Visualizations
- 🌍 Global Trends Analysis
- ✈️ Airline Performance Insights
- ⏳ Flight Delays & Statistics
- 📍 Airport Statistics

🔎 Start exploring now and gain deeper insights into aviation trends!

<div class="cards-container">
  <div class="card"><a href="1global-trends.html">🌍 Global Trends</a></div>
  <div class="card"><a href="2airline-performance.html">✈️ Airline Performance</a></div>
  <div class="card"><a href="3flight-delays.html">⏳ Flight Delays</a></div>
  <div class="card"><a href="4airport-statistics.html">📍 Airport Statistics</a></div>
</div>

<br>
<br>
<br>
<br>

## Dataset Overview: Flight Delay and Cancellation Data (2019–2023)
<div>

The foundation of this project is a curated subset of the publicly available Flight Delay and Cancellation Dataset (2019–2023), compiled by Patrick Zelazko and originally sourced from the U.S. Department of Transportation’s Bureau of Transportation Statistics (BTS). This dataset is publicly accessible via [Kaggle](https://www.kaggle.com/datasets/patrickzel/flight-delay-and-cancellation-dataset-2019-2023), providing researchers, analysts, and enthusiasts with a valuable resource for exploring flight performance metrics.
While the full dataset contains over 29 million records, our analysis focuses on a representative subset of 99,695 flights selected across various time frames, airlines, and airport routes between January 2019 and August 2023.

By focusing on a representative sample, the project retains the richness of the original data — covering multiple years, airlines, airports, and flight outcomes — while keeping the experience fast, smooth, and accessible for users.

Importantly, this subset is statistically and structurally diverse, meaning it still supports the extraction of valid and generalizable insights. It enables clear identification of trends, comparison of airline and airport performance, and an understanding of the causes and distribution of delays and cancellations — all without the noise and complexity of the full-scale dataset.

### Data Structure and Variables
Each flight record in the dataset includes 32 structured variables, capturing operational details from scheduling to performance outcomes. The key attributes used in our analysis can be grouped into the following categories:

🗓 **Temporal Information**
- FlightDate: The scheduled calendar date of the flight (YYYY-MM-DD format).

- Year, Month, DayOfWeek: Useful for seasonal and temporal trend analysis.

🛫 **Flight Details**
- Airline: The marketing carrier operating the flight (e.g., DL for Delta, AA for American Airlines).

- FlightNumber: Identifies the specific flight.

- Origin, Destination: IATA airport codes for departure and arrival.

- Distance: Great-circle distance between the two airports, in miles.

- Cancelled, Diverted: Boolean fields indicating if a flight was canceled or diverted.

⏱ **Scheduled and Actual Times**
- ScheduledDepartureTime, ScheduledArrivalTime: Planned takeoff and landing times.

- ActualDepartureTime, ActualArrivalTime: Real departure and arrival times (if not canceled).

- TaxiOut, TaxiIn: Time spent on the tarmac before takeoff or after landing.

- WheelsOff, WheelsOn: Exact airborne and touchdown times.

- ElapsedTime: Total time in the air.

⏳ **Delay Metrics**
- DepartureDelay, ArrivalDelay: Minutes late (or early) from schedule.

- DelayCarrier, DelayWeather, DelayNAS, DelaySecurity, DelayLateAircraft: Delay attribution categories explaining how much time was lost to each cause.

❌ **Cancellation Reasons**
- CancellationCode: A short categorical code(Carrier, Weather, National Airspace System (NAS), Security)

- Cancelled: Set to 1 if the flight was canceled, 0 otherwise.

These variables are used throughout the project to explore how flights behave over time, which airlines perform better, how airports differ in reliability, and what causes disruptions in the system.

</div><br><br>

<div>

## Objective of the Analysis

An important practical objective of this project is to **guide users in making better-informed decisions when selecting airports for travel**. By leveraging detailed airport-level metrics, such as average delays, delay variability, and rates of cancellations and diversions, the analysis enables users to:

- **Identify airports with consistently low delays**, making them more reliable for tight schedules or connections.
- **Avoid airports with high variability in performance**, especially during peak travel seasons.
- **Compare performance among major hubs**, which is especially useful when users have options between nearby alternatives (e.g., flying from JFK vs. Newark or LAX vs. Burbank).
- **Understand temporal performance trends** (e.g., certain airports might perform better in summer but worse in winter due to weather-related delays).

The **box plots**, **radar charts**, and **bubble maps** in the project are particularly geared toward highlighting these differences, empowering travelers with data to **reduce risk and plan smarter**.

Another goal of this analysis is to **uncover patterns and insights** related to flight delays and cancellations within the U.S. domestic airline industry over the specified period. By leveraging the rich dataset, the analysis also aims to:

1. **Identify Temporal Trends:** Examine how flight delays and cancellations have evolved over time, particularly in response to significant events like the COVID-19 pandemic.

2. **Assess Airline Performance:** Compare different airlines based on their delay and cancellation rates, providing a benchmark for operational efficiency.

3. **Evaluate Airport Efficiency:** Analyze airport-specific data to determine which airports experience higher rates of delays or cancellations, potentially highlighting infrastructural or logistical challenges.

4. **Understand Delay Causes:** Delve into the reasons behind flight delays, distinguishing between factors such as weather, carrier-related issues, and air traffic control constraints.

5. **Explore Temporal and Seasonal Patterns:** Investigate how delays vary by time of day, day of the week, and season, offering insights into peak periods and potential scheduling optimizations.

6. **Inform Stakeholders:** Provide actionable insights for airlines, airport authorities, policymakers, and travelers to improve decision-making and enhance the overall efficiency of air travel.

</div><br>

<footer style="display: flex; justify-content: space-between; align-items: center; padding: 10px;">
  <p>Built with <a href="https://observablehq.com/" target="_blank">Observable</a>.</p>
  <p>Developed by Elena Martino & Elisa Calza</p>
</footer>
