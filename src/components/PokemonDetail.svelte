<script>
   import { onMount } from "svelte";
   import { pokemons } from '../store/store';

   export let selectedPokemon;
   export let onBack;
   export let onSelect;

   let pokemonEvolutions = null;
   let selectedAbility = '';
   let loadingPokemonsAbility = false;

   function volver() {
      if (onBack) onBack();
   }

   function capitalize(str) {
      return str.charAt(0).toUpperCase() + str.slice(1);
   }

   async function loadEvolution(){
      const evolutions = await fetch(selectedPokemon.evolutions.url);
      const evolutionsData = await evolutions.json();
      const evolutionChain = evolutionsData.evolution_chain.url;
      const evolutionResponse = await fetch(evolutionChain);
      const evoData = await evolutionResponse.json();
      return evoData.chain;
   }

   function parseEvolutionChain(chain) {
      const evolutions = [];
      let current = chain;
      while(current) {
         evolutions.push(current.species.name);
         current = current.evolves_to[0] || null;
      }
      return evolutions;
   }

   async function goToEvolution(name) {
      if (name === selectedPokemon.name) return;
      const response = await fetch(`https://pokeapi.co/api/v2/pokemon/${name}`);
      const p = await response.json();

      const pokemonData = {
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
      };

      if (onSelect) onSelect(pokemonData);
   }


   function verDetallePokemon(pokemon) {
      selectedAbility = '';
      if (onSelect) onSelect(pokemon);
   }

   function handleSelectAbility(ability) {
      loadingPokemonsAbility = true;
      selectedAbility = ability;

      setTimeout(() => {
         loadingPokemonsAbility = false;
      }, 300);
   }

   onMount(async () => {
      const chain = await loadEvolution();
      pokemonEvolutions = parseEvolutionChain(chain);
   });

   $: if (selectedPokemon) {
      loadEvolution().then(chain => {
         pokemonEvolutions = parseEvolutionChain(chain);
      });
   }

   $: pokemonsAbilities = $pokemons.filter(p => p.abilities.includes(selectedAbility) && selectedPokemon.name !== p.name);

   const typeColors = {
      grass: "bg-green-200 text-green-800",
      fire: "bg-red-200 text-red-800",
      water: "bg-blue-200 text-blue-800",
      bug: "bg-lime-200 text-lime-800",
      normal: "bg-gray-200 text-gray-800",
      electric: "bg-yellow-200 text-yellow-800",
      ground: "bg-yellow-300 text-yellow-900",
      poison: "bg-purple-200 text-purple-800",
      fairy: "bg-pink-200 text-pink-800",
      fighting: "bg-orange-200 text-orange-800",
      psychic: "bg-pink-300 text-pink-900",
      rock: "bg-yellow-400 text-yellow-900",
      ghost: "bg-indigo-200 text-indigo-800",
      ice: "bg-blue-100 text-blue-800",
      dragon: "bg-purple-300 text-purple-900",
      steel: "bg-gray-300 text-gray-900",
      dark: "bg-gray-800 text-white"
   };
</script>

<div class="max-w-3xl mt-8 mx-auto p-6 bg-white rounded-xl shadow">
   <div class="flex flex-col md:flex-row gap-6">
      <div class="flex justify-center md:justify-start">
         <img
            src={selectedPokemon.image}
            alt={selectedPokemon.name}
            class="w-40 h-40 object-contain"
         />
      </div>
      <div class="flex-1">
         <h2 class="text-3xl font-bold capitalize mb-2">{selectedPokemon.name}</h2>
         <div class="grid grid-cols-2 gap-2 text-sm text-gray-800 mb-4">
            <div><strong>Altura:</strong> {selectedPokemon.height}</div>
            <div><strong>Peso:</strong> {selectedPokemon.weight}</div>
            <div><strong>Experiencia:</strong> {selectedPokemon.baseExperience}</div>
         </div>

         <div class="mb-3">
            <h3 class="font-semibold">Tipos:</h3>
            <div class="flex flex-wrap gap-2 mt-1 justify-center">
               {#each selectedPokemon.types as type}
                  <span class={`px-2 py-1 rounded capitalize text-sm ${typeColors[type] || 'bg-gray-200 text-gray-800'}`}>
                     {capitalize(type)}
                  </span>
               {/each}
            </div>
         </div>

         <div class="mb-3">
            <h3 class="font-semibold">Habilidades:</h3>
            <div class="flex flex-wrap gap-2 mt-1 justify-center">
               {#each selectedPokemon.abilities as ability}
                  <button on:click={() => handleSelectAbility(ability)} class="inline-block bg-gray-100 text-gray-800 px-2 py-1 rounded capitalize text-sm">
                     {capitalize(ability)}
                  </button>
               {/each}
            </div>
         </div>

         <div class="mb-3">
            <h3 class="font-semibold">Stats:</h3>
            <div class="grid grid-cols-2 gap-1 mt-1">
               {#each selectedPokemon.stats as stat}
                  <div class="text-sm">
                     {capitalize(stat.name)}: <strong>{stat.base}</strong>
                  </div>
               {/each}
            </div>
         </div>
      </div>
   </div>

   <div class="mt-6">
      <h3 class="font-semibold mb-2 text-lg">Evoluciones</h3>
      {#if pokemonEvolutions && pokemonEvolutions.length > 1}
         <div class="flex flex-wrap gap-3 justify-center">
            {#each pokemonEvolutions as evo}
               <button
                  on:click={() => goToEvolution(evo)}
                  class="px-3 py-1 rounded capitalize transition disabled:opacity-50 disabled:cursor-not-allowed"
                  style="background-color: darkkhaki; color: #fff;"
                  disabled={evo === selectedPokemon.name}
               >
                  {capitalize(evo)} {evo === selectedPokemon.name ? '(Actual)' : ''}
               </button>
            {/each}
         </div>
      {:else if pokemonEvolutions}
         <p class="text-gray-600">Este Pokémon no tiene evoluciones.</p>
      {:else}
      <div class="flex justify-center items-center h-40">
         <div class="mb-5 spinner w-10 h-10 rounded-full border-4 border-gray-300 border-t-blue-500"></div>
         <p class="text-gray-600">Cargando evoluciones...</p>
      </div>
      {/if}
   </div>
</div>

{#if selectedAbility !== ''}
   {#if loadingPokemonsAbility}
   <div class="flex justify-center items-center h-40">
      <div class="spinner w-10 h-10 rounded-full border-4 border-gray-300 border-t-blue-500"></div>
   </div>
   {:else}
   <p class="mb-4 mt-4 text-gray-600">Mostrando {pokemonsAbilities.length} pokémon(s)</p>

   <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-6">
      {#each pokemonsAbilities as pokemon}
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
   {/if}
{/if}

<button
   on:click={volver}
   class="mt-8 px-4 py-2 buttonCyan"
>
   ← Volver al listado
</button>
