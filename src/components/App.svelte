<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { feature } from 'topojson-client';
  import { writable } from 'svelte/store';
  

  let data = [];
  let sortOrder;
  let selectedCountryID = null;
  let container;
  let chartGroup;
  let legendGroup;
  let selectedYear = 2000;
  let allowedYears = [2000, 2005, 2010, 2015, 2019, 2020, 2021];
  let filteredData = [];
  let tooltipContent = '';
  let tooltipStyle = 'visibility: hidden; position: absolute;';
  let yearIndex = allowedYears.indexOf(selectedYear);
  let countryInput = ''; // This variable will be updated with the input value
  let searchError = writable(false);
  let countries;
  let projection;
  let width, height;
  let zoom;
  let scale;
  let margin;
  let searchInput = writable('');
  let suggestions = writable([]);

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
    defineBlurFilter();

    let selectedCountryID = null; // Store the selected country's data
    const worldMapResponse = await fetch('https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json');
    const worldMap = await worldMapResponse.json();
    countries = feature(worldMap, worldMap.objects.countries).features;

    margin = {top: 20, right: 20, bottom: 20, left: 20},
    width = 960 - margin.left - margin.right,
    height = 500 - margin.top - margin.bottom;

    projection = d3.geoNaturalEarth1()
      .scale(230)
      .translate([width / 2, height / 1.8]);
    const path = d3.geoPath().projection(projection);

    chartGroup.selectAll("*").remove();

    const colorScale = d3.scaleSequential(d3.interpolateYlGnBu)
                         .domain([0, d3.max(filteredData, d => d.Value)]);

    zoom = d3.zoom()
      .scaleExtent([1, 8])
      .on('zoom', ({ transform }) => {
        chartGroup.attr('transform', transform);
        legendGroup.attr('transform', transform); // Apply the same transform to legend
      });
    
    container.on("click", function(event) {
      // This checks if the click was directly on the container and not on any child elements
      if (event.target.classList.contains("container-svg")) {
        resetZoomAndBlur();
      }
    }, true); 
    container.call(zoom);

    chartGroup.attr('viewBox', `0 0 ${width + margin.left + margin.right} ${height + margin.top + margin.bottom}`)
       .selectAll('path')
       .data(countries)
       .enter().append('path')
       .merge(chartGroup.selectAll('path'))
       .attr('d', path)
       .attr('fill', d => {
         const countryData = filteredData.find(fd => fd['Region/Country/Area name'] === d.properties.name);
         return countryData ? colorScale(countryData.Value) : '#ccc';
       })
        // Only apply the blur filter if there is a selected country and the current country is not the selected one
       .attr("filter", d => selectedCountryID && d.properties.id !== selectedCountryID ? "url(#blur)" : null)
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
       })
       .on('click', function(event, d){
          selectedCountryID = d.properties.id;
          const countryName = d.properties.name;
          const countryData = data.filter(item => item['Region/Country/Area name'] === countryName);

          // Calculate where to position the line plot
          const bounds = path.bounds(d);
          const plotX = bounds[0][10];
          const plotY = bounds[0][1]; 

          drawLinePlot(countryData, countryName, {x: plotX, y: plotY});
          // Apply blur filter to all countries
          chartGroup.selectAll('path').attr("filter", "url(#blur)");
          // Remove filter from the clicked country to focus
          d3.select(this).attr("filter", null);
          // existing zoom functionality here
          clicked(event, d);
          event.stopPropagation();
       })
       
    function defineBlurFilter() {
        // Define the blur filter
        container.select("defs").remove(); // Remove existing defs to avoid duplicates
        const defs = container.append("defs");
        const filter = defs.append("filter")
          .attr("id", "blur")
          .attr("x", "-20%")
          .attr("y", "-20%")
          .attr("width", "140%")
          .attr("height", "140%");
        filter.append("feGaussianBlur")
          .attr("in", "SourceGraphic")
          .attr("stdDeviation", "5");
      }
  }

  function drawLinePlot(countryData, countryName, position) {
      // Define dimensions and margins for the plot
      const margin = { top: 10, right: 30, bottom: 30, left: 60 },
            width = 400 - margin.left - margin.right,
            height = 300 - margin.top - margin.bottom;

      // Select the container and position it
      const container = d3.select("#linePlotContainer")
                          .style("right", `${position.x}px`)
                          .style("top", `${position.y + 150}px`);

      // Clear previous SVG to ensure only one plot is visible at a time
      container.select("svg").remove();

      // Append a new SVG to the container for the line plot
      const svg = container.append("svg")
                    .attr("width", width + margin.left + margin.right)
                    .attr("height", height + margin.top + margin.bottom);

      // Append a white background rect with a stroke and set opacity to 50%
      svg.append("rect")
        .attr("width", width + margin.left + margin.right)
        .attr("height", height + margin.top + margin.bottom)
        .attr("fill", "white")
        .attr("opacity", 0.7); // Set the opacity to 50%

      // Append group for the plot (this must come after the background rect)
      const plotGroup = svg.append("g")
                          .attr("transform", `translate(${margin.left},${margin.top})`);

      // Add X axis
      const x = d3.scaleLinear()
                  .domain(d3.extent(countryData, d => d.Year))
                  .range([0, width]);
      plotGroup.append("g")
              .attr("transform", `translate(0, ${height})`)
              .call(d3.axisBottom(x).tickFormat(d3.format("d")))
              .selectAll("text") // Select the text elements for the tick labels
              .style("font-weight", "bold"); // Make the text bolder

      // Add Y axis
      const y = d3.scaleLinear()
                  .domain([0, d3.max(countryData, d => d.Value)])
                  .range([height, 0]);
      plotGroup.append("g")
              .call(d3.axisLeft(y))
              .selectAll("text")
              .style("font-weight", "bold");

      // Add the line
      plotGroup.append("path")
              .datum(countryData)
              .attr("fill", "none")
              .attr("stroke", "steelblue")
              .attr("stroke-width", 1.5)
              .attr("d", d3.line()
                            .x(d => x(d.Year))
                            .y(d => y(d.Value)));

      // Add title
      plotGroup.append("text")
              .attr("x", (width / 2))             
              .attr("y", 8 - (margin.top / 2))
              .attr("text-anchor", "middle")  
              .style("font-size", "16px") 
              .style("text-decoration", "underline")  
              .text(`${countryName}`);

      // Highlight the current year and value
      const currentYearData = countryData.find(d => d.Year === selectedYear);
      if (currentYearData) {
          plotGroup.append("circle")
                  .attr("cx", x(currentYearData.Year))
                  .attr("cy", y(currentYearData.Value))
                  .attr("r", 5)
                  .attr("fill", "red");

          plotGroup.append("text")
                  .attr("x", x(currentYearData.Year) + 5)
                  .attr("y", y(currentYearData.Value) - 5)
                  .text(`${currentYearData.Value}`);
      }
  }

  function clicked(event = null, d) {
        // Stop the event from bubbling up
        if (event) event.stopPropagation();

        // Calculate bounds for the clicked country
        const bounds = d3.geoPath().projection(projection).bounds(d);
        const [[x0, y0], [x1, y1]] = bounds;

        // Log each value to make sure they are numbers
        console.log('x0:', x0, 'y0:', y0, 'x1:', x1, 'y1:', y1);

        // Calculate the center of the bounds
        const xCenter = (x0 + x1) / 2;
        const yCenter = (y0 + y1) / 2;
        console.log('xCenter:', xCenter, 'yCenter:', yCenter);
        // Calculate scale and translation to center the clicked country
        const dx = x1 - x0;
        const dy = y1 - y0;
        scale = Math.min(8, 0.9 / Math.max(dx / width, dy / height));
        const translate = [width / 2 - scale * xCenter, height / 2 - scale * yCenter];
        console.log('Scale:', scale, 'Translate:', translate, 'dx:', dx, 'dy:', dy, 'height:',height,'width:',width);
        // Apply the zoom transformation
        container.transition().duration(750).call(
            zoom.transform,
            d3.zoomIdentity.translate(translate[0], translate[1]).scale(scale)
        );
        // Remove any existing country name text
        chartGroup.selectAll(".country-name").remove();
        // Append a new text element for the clicked country's name
        // Adjust the x, y to the center of the country or a fixed position as needed
        chartGroup.append("text")
            .attr("class", "country-name")
            .attr("x", xCenter) // Use the center or a fixed position
            .attr("y", yCenter) // Adjust position based on zoom or preferences
            .attr("text-anchor", "middle")
            .attr("dy", "0.35em")
            .text(d.properties.name)
            .style("font-size", "10px") // Consider adjusting based on zoom level
            .style("fill", "black");
    }

  async function handleSearch() {
    const countryName = countryInput.toLowerCase(); // Assuming countryInput is your search input
    const country = countries.find(d => d.properties.name.toLowerCase() === countryName);
    if (country) {
    // Clear the line plot and country name text if they exist
    d3.select("#linePlotContainer").selectAll("*").remove(); 
    chartGroup.selectAll(".country-name").remove();
    // Remove blur from all countries
    chartGroup.selectAll('path').attr("filter", null);
    // Here we pass an empty event and the country data to the 'clicked' function
    clicked(null, country); 

    // Reset the input and hide the error message
    countryInput = '';
    searchError.set(false);
    } else {
    // Display the error message
    searchError.set(true);
    }
  }

  function resetZoomAndBlur() {

    // Remove blur from all countries
    chartGroup.selectAll('path').attr("filter", null);

    // Reset zoom with a transition
    container.transition().duration(750).call(
        zoom.transform,
        d3.zoomIdentity // This resets the zoom to the initial state correctly
    );
    
    // Clear the line plot and country name text if they exist
    d3.select("#linePlotContainer").selectAll("*").remove(); 
    chartGroup.selectAll(".country-name").remove();
  }

  function updateYear(event) {
    yearIndex = +event.target.value;
    selectedYear = allowedYears[yearIndex];
    drawChart();
  }

  function bindRankingListClicks() {
    d3.selectAll(".rankingitem").on("click", function(event, d) {
      // 'd' here refers to the data bound to the ranking item, which should include the country name
      const countryName = d['Region/Country/Area name'];

      // Find the corresponding country path in the SVG by matching the country name
      const countryPath = chartGroup.selectAll('path')
                            .data().find(pathData => pathData.properties.name === countryName);

      if (countryPath) {
        clicked(null, countryPath);
        // Remove blur from all countries
        chartGroup.selectAll('path').attr("filter", null);
        // Clear the line plot and country name text if they exist
        d3.select("#linePlotContainer").selectAll("*").remove(); 
      }
    });
  }

  function updateSuggestions(input) {
    const matchingCountries = data
      .filter(d => d['Region/Country/Area name'].toLowerCase().startsWith(input.toLowerCase()))
      .map(d => d['Region/Country/Area name']);
    suggestions.set(matchingCountries);
  }

  $: if (countryInput) {
    updateSuggestions(countryInput);
  } else {
    suggestions.set([]); // Clear suggestions if input is empty
  }

  function updateRanking(year) {
    // Filter the data for the selected year and sort it based on the value
    let rankedData = data.filter(d => d.Year === year)
                        .sort((a, b) => {
                          if (sortOrder === 'ascending') {
                            return d3.ascending(a.Value, b.Value);
                          } else {
                            return d3.descending(a.Value, b.Value);
                          }
                        });
    renderRanking(rankedData);
  }

  function renderRanking(rankedData) {
  // Select the ranking list container
  const rankingList = d3.select("#rankingList");
  rankingList.html(""); // Clear existing ranking list

  // Create list items for each country
  rankedData.forEach((country, index) => {
    rankingList.append("div")
      .datum(country) // Bind the country data to each ranking item
      .attr("class", "rankingitem")
      .text(`${index + 1}. ${country['Region/Country/Area name']}: ${country.Value}`);
  });

  bindRankingListClicks();
}

  function toggleSortOrder() {
    sortOrder = sortOrder === 'descending' ? 'ascending' : 'descending';
    updateRanking(selectedYear); // Call the ranking update function whenever the sort order changes
  }

  function selectCountry(countryName) {
  countryInput = countryName;
  // Optionally, trigger search or other action here
  // For example, you might want to automatically search when a suggestion is clicked
  handleSearch(); // Assuming this function can handle searching based on `countryInput`
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
      .attr('x', -10)
      .attr('y', 505)
      .text('Percentage of individuals using the internet');

    const legendX = -9;
    const legendY = 335;
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
    <!-- Add 'Year' prefix before displaying selectedYear -->
    <span class="year-label">Year {selectedYear}</span>
  </div>
  <input type="text" bind:value={countryInput} placeholder="Enter country name">
  {#if $suggestions.length}
  <ul class="suggestions-list">
    {#each $suggestions as suggestion}
    <li on:click={() => selectCountry(suggestion)}>{suggestion}</li>
    {/each}
  </ul>
  {/if}
  <button on:click={handleSearch}>Search</button>
  {#if $searchError}<div style="color: red;">No such country.</div>{/if} 
  <div id="rankingList" class="ranking-container"></div>
  <div id="linePlotContainer" style="position: absolute; z-index: 1000;"></div>
  <button on:click={toggleSortOrder}>Global Year Ranks</button> 
  <div class="graph-container">
    <svg class="container-svg" width="1090" height="600">
      <g transform="translate(150, 50)">
        <g class="chart-group"></g>
        <g class="legend-group"></g>
      </g>
    </svg>
    <div class="tooltip" style="{tooltipStyle}">{tooltipContent}</div>
    <svg id="linePlotSvg"></svg>
  </div>
  <div class="source-annotation">Source: UNdata</div>
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
    justify-content: left;
  }

.ranking-container {
  position: absolute;
  padding: 10px;
  height: 500px; /* Fixed height to enable scrolling */
  overflow-y: auto; /* Enables vertical scrolling */
  margin: 20px 0;
  border: 8px solid rgb(9, 73, 150);
  right: 0;
  top: 140px; /* Added 'px' to ensure proper positioning */
  width: 200px; /* Fixed width */
  background-color: rgba(245, 247, 247, 0.925);
  color: #01114a; /* Text color */
  box-shadow: 0px 4px 8px rgba(0,0,0,0.1); /* Optional: Adds a subtle shadow for depth */
  border-radius: 5px; /* Optional: Rounds the corners */
  z-index: 100; /* Ensures it's on top of other content if needed */
}

:global(.rankingitem) {
  cursor: pointer;
  padding: 5px;
  border: 1px solid rgb(28, 61, 143);
  border-bottom: 1px solid #ccc;
  color: #03213e; /* Default dark blue color for the text */
}

.suggestions-list {
  list-style-type: none;
  padding: 0;
  margin: 0;
  border-radius: 4px;
  max-height: 200px;
  overflow-y: auto;
  z-index: 100;
  background-color: rgba(245, 247, 247, 0.925);
  background-color: var(--color-bg);
  border: 1px solid var(--color-outline);
  box-shadow: 0 0 4px var(--color-shadow);

}

.suggestions-list li {
  padding: 8px;
  cursor: pointer;
}

.suggestions-list li:hover {
  background-color: #f0f0f0;
}

.graph-text {
  position: absolute; /* Position the element absolutely */
  bottom: -1400px; /* Adjust the distance from the bottom */
  width: 85%;
  left: 52%; /* Position it horizontally at the center */
  transform: translateX(-50%); /* Center the element horizontally */
  text-align: left; /* Align the text in the center */
  font-size: 16px; /* Adjust the font size as needed */
  color: #666; /* Adjust the color of the text */
}

</style>