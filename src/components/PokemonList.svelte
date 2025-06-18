<script>
  import { onMount } from 'svelte';
  import { pokemons, abilities } from '../store/store';
  import { typeColors } from "./typeColors";
  import * as d3 from 'd3';

  export let onSelect;

  let selectedAbility = '';
  let selectedHeight = 0;
  let heights = Array.from({ length: 100 }, (_, i) => i + 1);
  let isLoading = true;
  let isLoadingMore = false;
  let limit = 20;
  let offset = 0;
  let selectedPokemons = new Set();
  let chartContainer;
  let showGrafic = false;

  function toggleSelection(pokemon) {
    if (selectedPokemons.has(pokemon)) {
      selectedPokemons.delete(pokemon);
    } else {
      selectedPokemons.add(pokemon);
    }
    selectedPokemons = new Set(selectedPokemons);
  }

  function clearFiltersPokemonsSelected(){
    selectedPokemons = new Set();
    showGrafic = false;
  }

  function verDetallePokemon(pokemon) {
    if (onSelect) onSelect(pokemon);
  }

  function capitalize(str) {
    return str.charAt(0).toUpperCase() + str.slice(1);
  }

  async function loadPokemons(addMore = false) {
    if (addMore) {
      isLoadingMore = true;
    } else {
      isLoading = true;
      offset = 0;
    }

    const res = await fetch(`https://pokeapi.co/api/v2/pokemon?limit=${limit}&offset=${offset}`);
    const data = await res.json();

    const pokemonsData = await Promise.all(
      data.results.map(pokemon =>
        fetch(pokemon.url).then(res => res.json())
      )
    );

    const cleanedPokemons = pokemonsData.map(p => ({
      name: p.name,
      height: p.height,
      weight: p.weight,
      image: p.sprites.front_default,
      abilities: p.abilities.map(a => a.ability.name),
      baseExperience: p.base_experience,
      evolutions: p.species,
      stats: p.stats.map(s => ({
        name: s.stat.name,
        base: s.base_stat,
      })),
      types: p.types.map(t => t.type.name)
    }));

    if (addMore) {
      pokemons.update(current => [...current, ...cleanedPokemons]);
      isLoadingMore = false;
    } else {
      pokemons.set(cleanedPokemons);
      isLoading = false;
      offset = 0;
    }

    const abilitySet = new Set();
    cleanedPokemons.forEach(p =>
      p.abilities.forEach(ab => abilitySet.add(ab))
    );
    abilities.set(Array.from(abilitySet));
  }

  function verMas() {
    offset += limit;
    loadPokemons(true);
  }

  function clearFilters(){
    selectedHeight = 0;
    selectedAbility = '';
    loadPokemons(false);
  }

  $: if (selectedAbility || selectedHeight !== 0) {
    loadPokemons(false);
  }

  onMount(() => {
    loadPokemons(false);
  });

  $: pokemonList = $pokemons;
  $: filteredPokemons = pokemonList.filter(p => {
    const matchAbility = selectedAbility ? p.abilities.includes(selectedAbility) : true;
    const matchHeight = selectedHeight !== 0 ? p.height === selectedHeight : true;
    return matchAbility && matchHeight;
  });
  $: total = filteredPokemons.length;

  // Reactividad para el array de seleccionados para usar en el gráfico
  $: selectedArray = Array.from(selectedPokemons);

  // Cada vez que cambia selectedArray, actualizamos gráfico
  $: {
    if (selectedArray.length > 0) {
      drawChart();
    } else if (chartContainer) {
      d3.select(chartContainer).selectAll('*').remove();
    }
  }

  function drawChart() {
    // Limpiar antes de dibujar
    d3.select(chartContainer).selectAll('*').remove();

    // Preparar datos: stats por pokemon
    // stats comunes: hp, attack, defense, special-attack, special-defense, speed
    const statsNames = ['hp', 'attack', 'defense', 'special-attack', 'special-defense', 'speed'];

    // Datos en formato [{pokemonName, statName, value}, ...]
    const data = [];
    selectedArray.forEach(pokemon => {
      statsNames.forEach(statName => {
        const stat = pokemon.stats.find(s => s.name === statName);
        if (stat) {
          data.push({ pokemon: capitalize(pokemon.name), stat: statName, value: stat.base });
        }
      });
    });

    // Dimensiones
    const margin = { top: 30, right: 30, bottom: 50, left: 50 };
    const width = 600 - margin.left - margin.right;
    const height = 300 - margin.top - margin.bottom;

    // Crear SVG
    const svg = d3.select(chartContainer)
      .append('svg')
      .attr('width', width + margin.left + margin.right)
      .attr('height', height + margin.top + margin.bottom)
      .style('background', '#f9f9f9')
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    // Escalas
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

    // Eje X
    svg.append('g')
      .attr('transform', `translate(0,${height})`)
      .call(d3.axisBottom(x0));

    // Eje Y
    svg.append('g')
      .call(d3.axisLeft(y));

    // Tooltip simple
    const tooltip = d3.select(chartContainer)
      .append('div')
      .style('position', 'absolute')
      .style('background', 'white')
      .style('padding', '5px 10px')
      .style('border', '1px solid #ccc')
      .style('border-radius', '4px')
      .style('pointer-events', 'none')
      .style('opacity', 0);

    // Grupos por stat
    const statGroups = svg.selectAll('g.statGroup')
      .data(statsNames)
      .enter()
      .append('g')
      .attr('class', 'statGroup')
      .attr('transform', d => `translate(${x0(d)},0)`);

    // Barras
    statGroups.selectAll('rect')
      .data(statName => {
        return data.filter(d => d.stat === statName);
      })
      .enter()
      .append('rect')
      .attr('x', d => x1(d.pokemon))
      .attr('y', d => y(d.value))
      .attr('width', x1.bandwidth())
      .attr('height', d => height - y(d.value))
      .attr('fill', d => color(d.pokemon))
      .on('mouseover', (event, d) => {
        tooltip.style('opacity', 1)
          .html(`<strong>${d.pokemon}</strong><br>${capitalize(d.stat)}: ${d.value}`)
          .style('left', (event.pageX + 10) + 'px')
          .style('top', (event.pageY - 28) + 'px');
      })
      .on('mouseout', () => {
        tooltip.style('opacity', 0);
      });

    // Leyenda
    const legend = svg.append('g')
      .attr('transform', `translate(0,${-margin.top / 2})`);

    legend.selectAll('rect')
      .data(selectedArray.map(p => capitalize(p.name)))
      .enter()
      .append('rect')
      .attr('x', (d, i) => i * 100)
      .attr('width', 15)
      .attr('height', 15)
      .attr('fill', d => color(d));

    legend.selectAll('text')
      .data(selectedArray.map(p => capitalize(p.name)))
      .enter()
      .append('text')
      .attr('x', (d, i) => i * 100 + 20)
      .attr('y', 12)
      .text(d => d)
      .style('font-size', '12px')
      .style('fill', '#333');
  }

  function toggleGrafic() {
    showGrafic = !showGrafic;
  }
</script>

<style>
  .spinner {
    border-top-color: #6366f1;
    animation: spin 0.8s linear infinite;
  }
  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }
  /* Tooltip styles */
  .tooltip {
    position: absolute;
    text-align: center;
    padding: 6px;
    font: 12px sans-serif;
    background: white;
    border: 1px solid #ccc;
    border-radius: 4px;
    pointer-events: none;
  }
</style>

<h2 class="text-3xl font-bold mt-6 mb-4">Listado de Pokémons</h2>

<div class="flex flex-col">
  <div class="mb-6">
    <label for="ability" class="block mb-2 font-semibold">Filtrar por habilidad</label>
    <select
      id="ability"
      bind:value={selectedAbility}
      on:change={() => (isLoading = true, setTimeout(() => (isLoading = false), 300))}
      class="w-full p-2 border border-gray-300 rounded"
    >
      <option value="">Ver todos los pokemons</option>
      {#each $abilities as ability}
        <option value={ability}>{capitalize(ability)}</option>
      {/each}
    </select>
  </div>
  <div class="mb-6">
    <label for="ability" class="block mb-2 font-semibold">Filtrar por altura</label>
    <select
      id="ability"
      bind:value={selectedHeight}
      on:change={() => (isLoading = true, setTimeout(() => (isLoading = false), 300))}
      class="w-full p-2 border border-gray-300 rounded"
    >
      <option value={0}>Ver todos los pokemons</option>
      {#each heights as height}
        <option value={height}>{height}</option>
      {/each}
    </select>
  </div>
  <div class="self-end">
    <button class="mb-5" style="background-color: crimson; color: #fff;" on:click={clearFilters} disabled={selectedAbility === '' && selectedHeight === 0}>Limpiar filtros</button>
  </div>
</div>

{#if isLoading}
  <div class="flex justify-center items-center h-40">
    <div class="spinner w-10 h-10 rounded-full border-4 border-gray-300 border-t-blue-500"></div>
  </div>
{:else}
  <p class="mb-4 text-gray-600">Mostrando {total} pokémon(s)</p>
  
  {#if selectedPokemons.size > 0}
    <div class="flex flex-col mt-6 bg-gray-50 p-4 rounded-xl shadow mb-5">
      <h4 class="font-semibold mb-2">Pokémon seleccionados:</h4>
      <div class="grid grid-cols-1 md:grid-cols-3 lg:grid-cols-6 gap-6">
        {#each Array.from(selectedPokemons) as pokemon}
          <div class="flex flex-col w-33">
            <p class={`px-2 py-1 rounded capitalize text-sm ${typeColors[pokemon.types[0]] || 'bg-gray-200 text-gray-800'}`}>{capitalize(pokemon.name)}</p>
            <img src={pokemon.image} alt={pokemon.name} class="w-20 h-20 mx-auto mb-2" />
          </div>
        {/each}
      </div>
      {#if showGrafic}
        <div bind:this={chartContainer} class="mt-6 mb-3 flex justify-center"></div>
      {/if}
      <div class="self-end flex w-xs">
        <button class="mt-3 mr-2" style="background-color: crimson; color: #fff;" on:click={clearFiltersPokemonsSelected}>Eliminar todos</button>
        <button on:click={toggleGrafic} class="mt-3 buttonCyan disabled:opacity-50 disabled:cursor-not-allowed" disabled={selectedPokemons.size < 2}>
          {showGrafic ? 'Cerrar gráfico' : 'Comparar stats'}
        </button>
      </div>
    </div>
  {/if}

  <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-6">
    {#each filteredPokemons as pokemon}
      <div class="p-4 bg-white rounded-xl shadow hover:shadow-md transition relative">
        <input
          type="checkbox"
          class="absolute top-2 left-2 w-4 h-4 rounded shadow rounded-xl shadow focus:outline-none"
          checked={selectedPokemons.has(pokemon)}
          on:change={() => toggleSelection(pokemon)}
        />
        <img src={pokemon.image} alt={pokemon.name} class="w-20 h-20 mx-auto mb-2" />
        <h3 class="text-lg font-bold text-center capitalize">{pokemon.name}</h3>
        <p class="text-sm text-center text-gray-500">Altura: {pokemon.height} | Peso: {pokemon.weight}</p>
        <div class="mt-2 text-xs text-center text-gray-600">
          {#each pokemon.abilities as ability}
            <span class="inline-block bg-gray-200 rounded px-2 py-1 mr-1 mt-1">{capitalize(ability)}</span>
          {/each}
          <button class="mt-3" style="background-color: darkkhaki; color: #fff;" on:click={() => verDetallePokemon(pokemon)}>Ver Pokémon</button>
        </div>
      </div>
    {/each}
  </div>

  {#if total !== 0}
    <div class="col-span-full flex flex-col items-center mt-4">
      {#if isLoadingMore}
        <div class="mt-5 spinner w-8 h-8 rounded-full border-4 border-gray-300 border-t-blue-500 mb-5"></div>
      {/if}
      <button on:click={verMas} class="buttonCyan" disabled={isLoadingMore}>Ver más..</button>
    </div>
  {/if}
{/if}
