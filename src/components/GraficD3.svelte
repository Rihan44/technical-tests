<script>
  import * as d3 from 'd3';
  // PROP de los pokemons seleecionados
  export let selectedPokemons = new Set();  

  let chartContainer;

  function capitalize(str) {
    console.log(selectedPokemons)
    return str.charAt(0).toUpperCase() + str.slice(1);
  }

  $: selectedArray = Array.from(selectedPokemons);
  console.log(selectedArray)

  $: {
    if (selectedArray.length > 0 && chartContainer) {
      drawChart();
    } else if (chartContainer) {
      d3.select(chartContainer).selectAll('*').remove();
    }
  }

  function drawChart() {
    d3.select(chartContainer).selectAll('*').remove();

    const statsNames = ['hp', 'attack', 'defense', 'special-attack', 'special-defense', 'speed'];
    const data = [];
    selectedArray.forEach(pokemon => {
      statsNames.forEach(statName => {
        const stat = pokemon.stats.find(s => s.name === statName);
        if (stat) {
          data.push({ pokemon: capitalize(pokemon.name), stat: statName, value: stat.base });
        }
      });
    });

    const margin = { top: 30, right: 30, bottom: 50, left: 50 };
    const width = 600 - margin.left - margin.right;
    const height = 300 - margin.top - margin.bottom;

    const svg = d3.select(chartContainer)
      .append('svg')
      .attr('width', width + margin.left + margin.right)
      .attr('height', height + margin.top + margin.bottom)
      .style('background', '#f9f9f9')
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    const x0 = d3.scaleBand()
      .domain(statsNames)
      .range([0, width])
      .paddingInner(0.1);

    const x1 = d3.scaleBand()
      .domain(selectedArray.map(p => capitalize(p.name)))
      .range([0, x0.bandwidth()])
      .padding(0.05);

    const y = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.value)]).nice()
      .range([height, 0]);

    const color = d3.scaleOrdinal()
      .domain(selectedArray.map(p => capitalize(p.name)))
      .range(d3.schemeCategory10);

    svg.append('g')
      .attr('transform', `translate(0,${height})`)
      .call(d3.axisBottom(x0));

    svg.append('g')
      .call(d3.axisLeft(y));

    const statGroups = svg.selectAll('g.statGroup')
      .data(statsNames)
      .enter()
      .append('g')
      .attr('class', 'statGroup')
      .attr('transform', d => `translate(${x0(d)},0)`);

    statGroups.selectAll('rect')
      .data(statName => data.filter(d => d.stat === statName))
      .enter()
      .append('rect')
      .attr('x', d => x1(d.pokemon))
      .attr('y', d => y(d.value))
      .attr('width', x1.bandwidth())
      .attr('height', d => height - y(d.value))
      .attr('fill', d => color(d.pokemon));
  }
</script>

<div bind:this={chartContainer} class="mt-6 mb-3 flex justify-center"></div>
