<script setup lang="ts">
import { ref, watch, onMounted, computed } from 'vue';
import axios from 'axios';

// Interfaces
interface Type {
  name: string;
  url: string;
}

interface PokemonDetailed {
  id: number;
  name: string;
  types: { slot: number; type: Type }[];
  sprites: {
    front_default: string;
    other: { 'official-artwork': { front_default: string } };
  };
  cries: { latest: string };
  abilities: { ability: { name: string }; is_hidden: boolean }[];
  stats: { base_stat: number; stat: { name: string } }[];
}

// --- State ---
const allPokemonOfType = ref<{ pokemon: { name: string; url: string } }[]>([]);
const pokemon = ref<PokemonDetailed[]>([]);
const totalPokemon = ref(0);
const loading = ref(true);
const limit = 12;
const types = ref<Type[]>([]);
const chosenTypes = ref<string[]>([]);
const page = ref(1);
const chosenPokemon = ref<PokemonDetailed | null>(null);
const pokemonModalOpen = ref(false);

// --- Helpers ---
const typeColors: { [key: string]: string } = {
	normal: '#A8A77A', fire: '#EE8130', water: '#6390F0', electric: '#F7D02C',
	grass: '#7AC74C', ice: '#96D9D6', fighting: '#C22E28', poison: '#A33EA1',
	ground: '#E2BF65', flying: '#A98FF3', psychic: '#F95587', bug: '#A6B91A',
	rock: '#B6A136', ghost: '#735797', dragon: '#6F35FC', dark: '#705746',
	steel: '#B7B7CE', fairy: '#D685AD',
};
const zeroPad = (num: number, places: number) => String(num).padStart(places, '0');
const capitalizeFirstLetter = (string: string) => string.charAt(0).toUpperCase() + string.slice(1);

// A computed property to generate the page title dynamically
const pageTitle = computed(() => {
  if (totalPokemon.value === 0 && !loading.value) return "No pokemons found";
  return `Showing ${pokemon.value.length} of ${totalPokemon.value} pokemons`;
});


// --- Data Fetching ---
async function fetchPokemonDetails(pokemonList: { pokemon: { name: string; url: string } }[]) {
  const promises = pokemonList.map(p => axios.get(`https://pokeapi.co/api/v2/pokemon/${p.pokemon.name}`));
  const responses = await Promise.all(promises);
  pokemon.value = responses.map(res => ({
    ...res.data,
    name: capitalizeFirstLetter(res.data.name),
  }));
}

async function fetchData() {
  loading.value = true;
  pokemon.value = [];

  if (chosenTypes.value.length === 0) {
    // Fetch the paginated list of pokemon
    const res = await axios.get(`https://pokeapi.co/api/v2/pokemon?limit=${limit}&offset=${(page.value - 1) * limit}`);
    totalPokemon.value = res.data.count;
    await fetchPokemonDetails(res.data.results.map((p: any) => ({ pokemon: p })));
  } else {
    // Filtered fetch
    let filteredList = [...allPokemonOfType.value]; // Use the pre-fetched list

    // Further filter if more than one type is selected
    for (let i = 1; i < chosenTypes.value.length; i++) {
        const typeRes = await axios.get(`https://pokeapi.co/api/v2/type/${chosenTypes.value[i]}`);
        const namesToKeep = new Set(typeRes.data.pokemon.map((p: any) => p.pokemon.name));
        filteredList = filteredList.filter(p => namesToKeep.has(p.pokemon.name));
    }

    totalPokemon.value = filteredList.length;
    const paginatedList = filteredList.slice((page.value - 1) * limit, page.value * limit);
    await fetchPokemonDetails(paginatedList);
  }

  loading.value = false;
}

// Watcher for when the type selection changes
watch(chosenTypes, async (newTypes) => {
    page.value = 1; // Reset page
    if (newTypes.length > 0) {
        loading.value = true;
        // Fetch all pokemon for the first selected type to enable client-side filtering for subsequent types
        const res = await axios.get(`https://pokeapi.co/api/v2/type/${newTypes[0]}`);
        allPokemonOfType.value = res.data.pokemon.filter((p: any) => {
            const id = p.pokemon.url.split('/').filter(Boolean).pop();
            return id && parseInt(id, 10) < 10000;
        });
        await fetchData();
    } else {
        // If all types are deselected, fetch the default list
        allPokemonOfType.value = [];
        await fetchData();
    }
}, { deep: true }); // Use deep watch for array changes

// Watcher for when the page changes
watch(page, fetchData);

// --- Lifecycle Hook ---
onMounted(async () => {
  const res = await axios.get(`https://pokeapi.co/api/v2/type?limit=18`);
  types.value = res.data.results;
  await fetchData(); // Initial fetch
});

// --- Event Handlers ---
function selectType(typeName: string) {
  const index = chosenTypes.value.indexOf(typeName);
  if (index > -1) {
    chosenTypes.value.splice(index, 1);
  } else {
    chosenTypes.value.push(typeName);
  }
}

function openPokemonModal(p: PokemonDetailed) {
  chosenPokemon.value = p;
  pokemonModalOpen.value = true;
}
</script>

<template>
  <v-app>
    <v-main>
      <v-container class="text-center page-content">
        <div class="framework-switcher">
          <div class="title-container">
            <img src="/pokemon-icon.webp" alt="Pokeball Icon" class="pokeball-icon" />
            <h1 class="h1 font-bold tracking-tight">Guido's Pokédex</h1>
          </div>
          <a href="https://guidos-pokedex-react.netlify.app" class="framework-link">React</a>|
          <span class="framework-link-active">Vue</span>
          |<a href="https://guidos-pokedex-svelte.netlify.app" class="framework-link">Svelte</a>
        </div>
        <h3 class="font-weight-medium mb-2 filter-by-text">Filter by type:</h3>
        <div class="pa-2 filter-buttons">
          <v-btn
            v-for="type in types"
            :key="type.name"
            :color="typeColors[type.name]"
            :variant="chosenTypes.includes(type.name) ? 'outlined' : 'text'"
            class="ma-1 uppercase filter-button font-weight-medium"
            @click="selectType(type.name)"
          >
            {{ type.name }}
          </v-btn>
        </div>
        <h4 class="number-of-pokemon">{{ pageTitle }}</h4>
        <v-row no-gutters>
          <v-col v-if="loading" v-for="n in limit" :key="`skel-${n}`" cols="6" sm="3" md="2" xl="1">
            <v-skeleton-loader class="ma-2" type="image, list-item-two-line"></v-skeleton-loader>
          </v-col>
          <v-col v-else v-for="p in pokemon" :key="p.id" cols="6" sm="3" md="2" xl="1">
            <v-card
              class="pokemon-card ma-2"
              variant="flat"
              :style="{ background: `linear-gradient(45deg, ${typeColors[p.types[0].type.name]} 85%${p.types[1] ? `, ${typeColors[p.types[1].type.name]} 85%` : ''})` }"
              @click="openPokemonModal(p)"
            >
              <div class="pokemon-card-content">
                <div class="pokemon-info">
                  <div class="pokemon-info-name">{{ p.name }}</div>
                  <div class="pokemon-info-id">#{{ zeroPad(p.id, 3) }}</div>
                </div>
                <img :src="p.sprites.front_default" class="pokemon-sprite" />
              </div>
              <img src="/pokeball.svg" class="pokeball-bg" />
            </v-card>
          </v-col>
        </v-row>
        
        <v-pagination
          v-if="!loading && totalPokemon > limit"
          v-model="page"
          :length="Math.ceil(totalPokemon / limit)"
          :total-visible="6"
          class="mt-3"
          rounded="circle"
          variant="flat"
          active-color="primary"
          density="comfortable"
          size="small"
        ></v-pagination>

        <v-dialog v-model="pokemonModalOpen" max-width="450" >
          <v-card v-if="chosenPokemon" class="rounded-xl overflow-hidden pokemon-modal">
            <v-card-title 
              :style="{ backgroundColor: typeColors[chosenPokemon.types[0].type.name] }" 
              class="pa-4 text-white d-flex align-center modal-header"
            >
              <div class="modal-pokemon-title">
                {{ chosenPokemon.name }}
                <span class="opacity-70 font-weight-regular ml-2">#{{ zeroPad(chosenPokemon.id, 3) }}</span>
              </div>
              <v-spacer></v-spacer>
              <v-btn icon="mdi-close" variant="text" @click="pokemonModalOpen = false"></v-btn>
            </v-card-title>
            <v-card-text class="pt-4">
              <v-img :src="chosenPokemon.sprites.other['official-artwork'].front_default" maxHeight="300" contain></v-img>
              <audio
                v-if="chosenPokemon.cries.latest"
                style="width: 100%;"
                :src="chosenPokemon.cries.latest"
                controls
                volume="0.05"
                autoplay
              ></audio>

              <v-row class="mt-4">
                <v-col cols="6">
                  <h4 
                    class="mb-1 modal-subtitle" 
                    :style="{ color: typeColors[chosenPokemon.types[0].type.name] }"
                  >
                    Base Stats
                  </h4>
                  <div v-for="stat in chosenPokemon.stats" :key="stat.stat.name" class="stat-row">
                    <span class="stat-name">{{ capitalizeFirstLetter(stat.stat.name.replace('-', ' ')) }}</span>
                    <span class="stat-value">{{ stat.base_stat }}</span>
                  </div>
                </v-col>
                <v-col cols="6">
                  <h4 
                    class="mb-1 modal-subtitle"
                    :style="{ color: typeColors[chosenPokemon.types[0].type.name] }"
                  >
                    Type
                  </h4>
                  <div class="types-list">
                    <v-chip v-for="t in chosenPokemon.types" :key="t.type.name" class="mb-1 type-chip text-md" variant="flat" density="compact" size="large" :color="typeColors[t.type.name]">
                      {{ capitalizeFirstLetter(t.type.name) }}
                    </v-chip>
                  </div>
                  
                  <h4 
                    class="mb-1 mt-4 modal-subtitle"
                    :style="{ color: typeColors[chosenPokemon.types[0].type.name] }"
                  >
                    Abilities
                  </h4>
                  <div v-for="ability in chosenPokemon.abilities" :key="ability.ability.name" class="ability-name">
                    {{ capitalizeFirstLetter(ability.ability.name) }}
                    <span v-if="ability.is_hidden" class="">(hidden)</span>
                  </div>
                </v-col>
              </v-row>
            </v-card-text>
            <v-divider></v-divider>
            <v-card-actions class="pa-2">
              <v-spacer></v-spacer>
              <v-btn 
                :color="typeColors[chosenPokemon.types[0].type.name]" 
                variant="text"
                @click="pokemonModalOpen = false"
              >
                Close
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>
      </v-container>
    </v-main>
  </v-app>
</template>

<style>

.page-content {
  max-width: 100% !important;
  padding: 24px 24px 0 24px !important;
}

.framework-switcher {
  padding: 16px;
  text-align: center;
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  margin-bottom: 24px;
}
.framework-link {
  color: #007bff;
  margin: 0 12px;
  text-decoration: none;
  font-size: 1.1rem;
}
.framework-link-active {
  /* color: black; */
  margin: 0 12px;
  text-decoration: none;
  font-size: 1.1rem;
  font-weight: bold;
  cursor: default;
}

.title-container {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 1rem;
  /* color: #333;  */
}

.title-container h1 {
  font-size: 2.5rem; /* 40px */
  font-weight: bold;
  margin: 0;
}

.pokeball-icon {
  width: 48px;
  height: 48px;
  animation: spin 8s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.filter-by-text {
  font-size: 1.5rem;
}

.filter-buttons {
  border: 1px solid lightgrey;
  border-radius: 16px;
}

.filter-button {
  font-weight: 500;
  letter-spacing: normal !important;
  padding: 6px 8px !important;
}

.number-of-pokemon {
  margin: 24px 0 12px 0;
  font-size: 2.125rem;
  font-weight: 500;
}

/* Scoped styles for the Pokemon Card */
.pokemon-card {
  border-radius: 16px !important;
  overflow: hidden;
  position: relative;
  cursor: pointer;
  /* box-shadow: 0 4px 12px rgba(0,0,0,0.1); */
  height: 192px;
  transition: transform 0.2s ease-in-out, box-shadow 0.2s ease-in-out;
}

.pokemon-card:hover {
  transform: scale(1.03);
  box-shadow: 0 0px 16px rgba(0,0,0,0.2);
}

.pokemon-card-content {
  position: relative;
  display: flex;
  flex-direction: column;
  /* justify-content: flex-end; */
}

.pokeball-bg {
  position: absolute !important;
  bottom: -74px;
  right: calc(-140px + 35%);
  width: 120%;
  height: 120%;
  opacity: 0.3;
  z-index: 2;
}

.pokemon-sprite {
  /* width: 100px; */
  height: 140px;
  width: inherit;
  max-width: 140px;
  margin: -8px;
  margin-top: -20px;
  margin-left: auto;
  z-index: 3;
}

.pokemon-info {
  text-align: left;
  position: relative;
  z-index: 4;
  padding: 0px 8px;
}
.pokemon-info-name {
  margin-top: 16px;
  /* font-weight: bold; */
  color: white;
  font-size: 1.5rem;
}
.pokemon-info-id {
  /* color: white; */
  font-weight: 500;
  font-size: 1.25rem;
  opacity: 0.4;
}

.modal-header {
  font-size: 1.75rem !important;
  font-weight: 500;
}

.modal-pokemon-title span {
  font-size: 1.3rem;
}

.modal-subtitle {
  font-size: 1.5rem;
  font-weight: 400;
}

.types-list {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.types-list .type-chip {
  color: white !important;
}

.stat-row {
  display: flex;
  justify-content: space-between;
  padding: 2px 0;
}

.ability-name {
  padding: 2px 0;
}
</style>