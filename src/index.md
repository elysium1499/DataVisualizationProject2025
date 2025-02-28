---
theme: dashboard
title: Flights Analysis
toc: true
---

<style>
/* Theme Styling */
:root {
  --theme-background: #121212;
  --theme-foreground: #ffffff;
  --theme-foreground-muted: #b0bec5;
  --theme-border: #444444;
  --theme-card-background: #1e1e1e;
}

body {
  background-color: var(--theme-background);
  color: var(--theme-foreground);
}

.navbar {
  display: flex;
  justify-content: center;
  gap: 1rem;
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
  text-align: center;
  margin: 2rem auto;
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
  border-radius: 8px;
  background-color: var(--theme-card-background);
  text-align: center;
  width: 200px;
}

.card a {
  color: white;
  text-decoration: none;
  font-weight: bold;
}

footer {
  text-align: center;
  padding: 1rem;
  margin-top: 2rem;
}
</style>

<div class="hero">
  <h1>Flights Analysis</h1>
  <h2>Explore flight data and insights with our interactive dashboard.</h2>
</div>

<nav class="navbar">
  <a href="index.html" class="active">🏠 Home</a>
  <a href="global-trends.html">🌍 Global Trends</a>
  <a href="airline-performance.html">✈ Airline Performance</a>
  <a href="airport-statistics.html">📍 Airport Statistics</a>
  <a href="flight-delays.html">⏳ Flight Delays</a>
</nav>

## Welcome to Flights Analysis

Welcome to **Flights Analysis**, a platform for exploring global flight data, trends, and insights.

### Why use this dashboard?
- 📊 **Interactive Data Visualizations**
- 🌍 **Global Trends Analysis**
- ✈ **Airline Performance Insights**
- 📍 **Airport Statistics**
- ⏳ **Flight Delays & Statistics**

Start exploring now!

<div class="cards-container">
  <div class="card"><a href="1global-trends.html">🌍 Global Trends</a></div>
  <div class="card"><a href="2airline-performance.html">✈ Airline Performance</a></div>
  <div class="card"><a href="3airport-statistics.html">📍 Airport Statistics</a></div>
  <div class="card"><a href="4flight-delays.html">⏳ Flight Delays</a></div>
</div>

<footer>
  <p>Built with <a href="https://observablehq.com/" target="_blank">Observable</a>.</p>
  <p>Developed by Elena Martino & Elisa Calza</p>
</footer>
