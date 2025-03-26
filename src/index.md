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
- ✈ Airline Performance Insights
- 📍 Airport Statistics
- ⏳ Flight Delays & Statistics

🔎 Start exploring now and gain deeper insights into aviation trends!

<div class="cards-container">
  <div class="card"><a href="1global-trends.html">🌍 Global Trends</a></div>
  <div class="card"><a href="2airline-performance.html">✈ Airline Performance</a></div>
  <div class="card"><a href="3flight-delays.html">⏳ Flight Delays</a></div>
  <div class="card"><a href="4airport-statistics.html">📍 Airport Statistics</a></div>
</div>

<footer style="display: flex; justify-content: space-between; align-items: center; padding: 10px;">
  <p>Built with <a href="https://observablehq.com/" target="_blank">Observable</a>.</p>
  <p>Developed by Elena Martino & Elisa Calza</p>
</footer>
