# My Dashboard – Project Brief

## What is this? 
 I want a single page analytics dashboard showing monthly business metrics like a Shopify Admin or Google Analytics view.

## Data
Generate a fake dataset as a JSON file (src/data/metrics.json).
12 months of Data (Jan-Dec 2025), each month containing
- revenue (Dollar amount, trending upward with some variation)
- visitors (number, seasonal pattern - higher in summer)
- conversions (percentage, fluctuates between 2-5%)
- orders (number, correlates loosely with visitors)

## Layout (Vuetify)
- v-app-bar at the top with the dashboard tittle and a month picker
- The month picker should default to showing all months
- When a specific month is selected, all cards and charts filter to that month. When "All" is selected, show the full year.
- Below the app bar: a row of four summary cards (v-card) showing the key metrics - revenue, visitors, conversions, orders
- Below tthe cards: a row of 2 charts
    - Left: Bar chart showing monthly revenue
    - Right: Line chart showing visitors over time
- Below thatt: One full-width area chart showing conversions trend
- Use v-container, v-row, v-col for responsive grid layout (Desktop, tablet, mobile breakpoints)

## Interactions
- Month picker in the app bar filters EVERYTHING - summary cards show that month's number, charts highlight or filter to that month
- When "All" is selected, summary cards show yearly totals/averages and charts show all 12 months
- Cards should show a small up/down arrow or color indicating change from the previous month

## Style
- Dark theme by default (Vuetify dark theme)
- Clean, Minimal, Lots of Whitespace
- Charts should use a cohesive color pallette - not rainbow
- Mobile responsive - cards stack on small screens

## Tech
- Vue 3 + TypeScript + Vuetify 3
- Chart.js via vue-chartjs for all charts
- Fake data from a local JSON file (no API calls)
- Single page - no routing needed for this app



