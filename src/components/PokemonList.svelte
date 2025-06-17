<script>
   import { onMount } from 'svelte';
   import { pokemons, abilities } from '../store/store';
   export let onSelect;

   let selectedAbility = '';
   let selectedHeight = 0;
   let heights = Array.from({ length: 100 }, (_, i) => i + 1);
   let isLoading = true;      
   let isLoadingMore = false; 
   let limit = 20;
   let offset = 0;

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
         isLoadingMore = true;
         pokemons.update(current => [...current, ...cleanedPokemons]);
      } else {
         isLoading = true;
         offset = 0;
         pokemons.set(cleanedPokemons);
      }

      const abilitySet = new Set();
      cleanedPokemons.forEach(p =>
         p.abilities.forEach(ab => abilitySet.add(ab))
      );
      abilities.set(Array.from(abilitySet));

      if (addMore) {
         isLoadingMore = false;
      } else {
         isLoading = false;
      }
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

  <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-6">
    {#each filteredPokemons as pokemon}
      <div class="p-4 bg-white rounded-xl shadow hover:shadow-md transition">
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

