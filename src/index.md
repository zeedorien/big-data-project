---
toc: false
---


<!DOCTYPE html>
<html>
<head>
    <title>Beyond the Tarmac: Predicting Airline Delay Causes</title>
</head>
<div class="hero">
  <h1>Beyond the Tarmac: Predicting Airline Delay Causes</h1>
  <h2>Shreyon Roy, Crystal Zhang, Dorien Zhang</h2>
</div>
</html>

---

# Abstract

Our visualizations explore the factors that cause commercial flight delays across the US, revealing patterns across geography, time of year, carriers, and delay cause.
The visualizations are built with Observable Framework, D3, and DuckDB, handling millions of rows efficiently in the browser. They are also fully responsive, with tooltips and hover interactions so that the data can be examined more precisely.


---



<style>

.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: var(--sans-serif);
  margin: 4rem 0 4rem;
  text-wrap: balance;
  text-align: center;
}

.hero h1 {
  margin: 1rem 0;
  padding: 1rem 0;
  max-width: none;
  font-size: 14vw;
  font-weight: 900;
  line-height: 1;
  background: linear-gradient(30deg, var(--theme-foreground-focus), currentColor);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.hero h2 {
  margin: 0;
  max-width: 34em;
  font-size: 20px;
  font-style: initial;
  font-weight: 500;
  line-height: 1.5;
  color: var(--theme-foreground-muted);
}

@media (min-width: 640px) {
  .hero h1 {
    font-size: 90px;
  }
}

</style>
