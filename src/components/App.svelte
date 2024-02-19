<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { feature } from 'topojson-client';

  let data = [];
  let container;
  let chartGroup;
  let legendGroup;
  let selectedYear = 2000;
  let allowedYears = [2000, 2005, 2010, 2015, 2019, 2020, 2021];
  let filteredData = [];
  let tooltipContent = '';
  let tooltipStyle = 'visibility: hidden; position: absolute;';
  let yearIndex = allowedYears.indexOf(selectedYear);

  onMount(async () => {
    const res = await fetch('Internet_usage_data.csv');
    const csv = await res.text();
    data = d3.csvParse(csv, d3.autoType);

    container = d3.select('.container-svg');
    chartGroup = container.select('.chart-group');
    legendGroup = container.select('.legend-group');

    filteredData = data.filter(d => d.Year === selectedYear);
    drawChart();
    drawLegend();
  });

  $: yearIndex = allowedYears.indexOf(selectedYear);
  $: if (data.length > 0 && selectedYear) {
    filteredData = data.filter(d => d.Year === selectedYear);
    drawChart();
  }

  async function drawChart() {
    if (data.length === 0 || !container.node()) return;

    const margin = { top: 20, right: 20, bottom: 20, left: 20 },
          width = 960 - margin.left - margin.right,
          height = 500 - margin.top - margin.bottom;

    const worldMapResponse = await fetch('https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json');
    const worldMap = await worldMapResponse.json();
    const countries = feature(worldMap, worldMap.objects.countries).features;

    const projection = d3.geoNaturalEarth1()
      .scale(230)
      .translate([width / 2, height / 1.8]);
    const path = d3.geoPath().projection(projection);

    chartGroup.selectAll("*").remove();

    const colorScale = d3.scaleSequential(d3.interpolateYlGnBu)
                         .domain([0, d3.max(filteredData, d => d.Value)]);

    const zoom = d3.zoom()
      .scaleExtent([1, 8])
      .on('zoom', ({ transform }) => {
        chartGroup.attr('transform', transform);
        legendGroup.attr('transform', transform); // Apply the same transform to legend
      });

    container.call(zoom);

    chartGroup.attr('viewBox', `0 0 ${width + margin.left + margin.right} ${height + margin.top + margin.bottom}`)
       .selectAll('path')
       .data(countries)
       .enter().append('path')
       .attr('d', path)
       .attr('fill', d => {
         const countryData = filteredData.find(fd => fd['Region/Country/Area name'] === d.properties.name);
         return countryData ? colorScale(countryData.Value) : '#ccc';
       })
       .on('mouseover', (event, d) => {
         const countryData = filteredData.find(fd => fd['Region/Country/Area name'] === d.properties.name);
         if (countryData) {
           tooltipContent = `Region: ${countryData['Region/Country/Area name']}, Value: ${countryData.Value}`;
         } else {
           tooltipContent = `Region: ${d.properties.name}, No Data`;
         }
         tooltipStyle = `visibility: visible; top: ${event.pageY}px; left: ${event.pageX}px;`;
       })
       .on('mousemove', (event) => {
         tooltipStyle = `visibility: visible; top: ${event.pageY}px; left: ${event.pageX}px;`;
       })
       .on('mouseout', () => {
         tooltipContent = '';
         tooltipStyle = 'visibility: hidden;';
       });
  }

  function updateYear(event) {
    yearIndex = +event.target.value;
    selectedYear = allowedYears[yearIndex];
    drawChart();
  }

  function drawLegend() {
    const colorScale = d3.scaleSequential(d3.interpolateYlGnBu)
                         .domain([0, d3.max(filteredData, d => d.Value)]);
    const legend = legendGroup;

    const legendData = colorScale.ticks(5).map(d => ({
      value: d,
      color: colorScale(d)
    }));

    const legendTitle = legend.append('text')
      .attr('class', 'legend-title')
      .attr('x', -100)
      .attr('y', 475)
      .text('Percentage of individuals using the internet');

    const legendX = -99;
    const legendY = 305;
    const legendItem = legend.selectAll('.legend-item')
      .data(legendData)
      .enter()
      .append('g')
      .attr('class', 'legend-item')
      .attr('transform', (d, i) => `translate(${legendX}, ${legendY + i * 25})`);

    legendItem.append('rect')
      .attr('width', 20)
      .attr('height', 20)
      .attr('fill', d => d.color)
      .attr('stroke', 'black');

    legendItem.append('text')
      .attr('x', 30)
      .attr('y', 15)
      .text(d => d.value);
  }
</script>

<main>
  <h1>How Does Internet Usage Evolve around the World During the Past Two Decades?</h1>
  <div class="slider-container">
    <input
      type="range"
      min="0"
      max="{allowedYears.length - 1}"
      step="1"
      bind:value="{yearIndex}"
      on:input="{updateYear}"
    />
    <span class="year-label">{selectedYear}</span>
  </div>
  <div class="graph-container">
    <svg class="container-svg" width="1290" height="600">
      <g transform="translate(150, 50)">
        <g class="chart-group"></g>
        <g class="legend-group"></g>
      </g>
    </svg>
    <div class="tooltip" style={tooltipStyle}>{tooltipContent}</div>
  </div>
</main>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Nunito:wght@300;400;700&display=swap');
  :root {
    --color-bg: #ffffff;
    --color-outline: #c2c2c2;
    --color-shadow: hsl(0, 0%, 0%, 0.1);
    --color-text: #3f4252;
    --color-bg-1: hsla(0, 0%, 0%, 0.2);
    --color-shadow-1: hsl(0, 0%, 96%);
  }

  *,
  *::before,
  *::after {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  .slider-container {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
    margin: 10px 0;
  }

  main {
    text-align: center;
    font-family: 'Nunito', sans-serif;
    font-weight: 300;
    line-height: 2;
    font-size: 15px;
    color: var(--color-text);
  }

  h1 {
    font-size: 2em;
    font-weight: 300;
    line-height: 2;
  }

  .chart-svg {
    width: 100%;
    height: auto;
    display: block;
    background-color: var(--color-bg);
  }

  .tooltip {
    position: absolute;
    padding: 5px;
    background: rgba(0, 0, 0, 0.75);
    color: white;
    border-radius: 5px;
    pointer-events: none;
    transition: opacity 0.3s;
  }

  .graph-container {
    position: relative;
    display: flex;
    justify-content: center;
  }
</style>
