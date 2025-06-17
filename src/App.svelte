<script>
  import './app.pcss';
  import PokemonList from './components/PokemonList.svelte';
  import PokemonDetail from './components/PokemonDetail.svelte';

  let view = 'list';
  let selectedPokemon = null;

  function verDetalle(pokemon) {
    selectedPokemon = pokemon;
    view = 'detail';
  }

  function volverListado() {
    view = 'list';
    selectedPokemon = null;
  }
</script>

<main>
  <h1>Pokedex</h1>

  {#if view === 'list'}
    <PokemonList onSelect={verDetalle}/>
  {/if}

  {#if view === 'detail' && selectedPokemon}
    <PokemonDetail selectedPokemon={selectedPokemon} onBack={volverListado} onSelect={(newPoke) => selectedPokemon = newPoke}/>
  {/if}
</main>
