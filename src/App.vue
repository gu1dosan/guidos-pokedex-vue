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
      <v-container>
        <h3 class="text-h6 font-weight-regular mb-4">Filter by type:</h3>
        <v-card variant="outlined" class="pa-2 mb-10">
          <v-btn
            v-for="type in types"
            :key="type.name"
            :color="typeColors[type.name]"
            :variant="chosenTypes.includes(type.name) ? 'flat' : 'text'"
            class="ma-1 text-capitalize"
            @click="selectType(type.name)"
          >
            {{ type.name }}
          </v-btn>
        </v-card>

        <h2 class="text-h4 font-weight-regular mb-6">{{ pageTitle }}</h2>

        <v-row>
          <v-col v-if="loading" v-for="n in limit" :key="`skel-${n}`" cols="6" sm="4" md="3" lg="2">
            <v-skeleton-loader type="image, list-item-two-line"></v-skeleton-loader>
          </v-col>
          <v-col v-else v-for="p in pokemon" :key="p.id" cols="6" sm="4" md="3" lg="2">
            <v-card
              class="pokemon-card"
              :style="{ backgroundColor: typeColors[p.types[0].type.name] }"
              @click="openPokemonModal(p)"
            >
              <div class="pokemon-card-content">
                <div class="pokemon-card-background" :style="{ background: `linear-gradient(45deg, ${typeColors[p.types[0].type.name]} 85%${p.types[1] && `, ${typeColors[p.types[1].type.name]} 85%`})`}">
                  <v-img src="/pokeball.svg" class="pokeball-bg" />
                </div>
                <v-img :src="p.sprites.front_default" class="pokemon-sprite" />
                <div class="pokemon-info">
                  <div class="font-weight-bold text-white">{{ p.name }}</div>
                  <div class="text-white text-caption">#{{ zeroPad(p.id, 3) }}</div>
                </div>
              </div>
            </v-card>
          </v-col>
        </v-row>
        
        <v-pagination
          v-if="!loading && totalPokemon > limit"
          v-model="page"
          :length="Math.ceil(totalPokemon / limit)"
          :total-visible="5"
          class="mt-10"
        ></v-pagination>

        <v-dialog v-model="pokemonModalOpen" max-width="500">
          <v-card v-if="chosenPokemon">
            <v-card-title :style="{ backgroundColor: typeColors[chosenPokemon.types[0].type.name] }" class="text-h5 pa-4 text-white">
              {{ chosenPokemon.name }}
              <span class="font-weight-light ml-2">#{{ zeroPad(chosenPokemon.id, 3) }}</span>
            </v-card-title>
            <v-card-text class="pt-4">
              <v-img :src="chosenPokemon.sprites.other['official-artwork'].front_default" height="250" contain></v-img>
              <audio
                v-if="chosenPokemon.cries.latest"
                style="width: 100%; margin-top: 1rem;"
                :src="chosenPokemon.cries.latest"
                controls
                volume="0.05"
                autoplay
              ></audio>

              <v-row class="mt-4">
                <v-col cols="6">
                  <h4 class="text-h6 mb-2">Stats</h4>
                  <div v-for="stat in chosenPokemon.stats" :key="stat.stat.name">
                    {{ capitalizeFirstLetter(stat.stat.name) }}: {{ stat.base_stat }}
                  </div>
                </v-col>
                <v-col cols="6">
                  <h4 class="text-h6 mb-2">Types</h4>
                  <v-chip v-for="t in chosenPokemon.types" :key="t.type.name" :color="typeColors[t.type.name]" class="mr-2 mb-2 text-capitalize">
                    {{ capitalizeFirstLetter(t.type.name) }}
                  </v-chip>
                  <h4 class="text-h6 mb-2 mt-4">Abilities</h4>
                  <div v-for="ability in chosenPokemon.abilities" :key="ability.ability.name">
                    {{ capitalizeFirstLetter(ability.ability.name) }}
                    <span v-if="ability.is_hidden">(Hidden)</span>
                  </div>
                </v-col>
              </v-row>
            </v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn color="primary" text @click="pokemonModalOpen = false">Close</v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>
      </v-container>
    </v-main>
  </v-app>
</template>

<style>
/* Global style override to make pagination buttons circular */
.v-pagination .v-btn {
  border-radius: 50% !important;
}

/* Scoped styles for the Pokemon Card */
.pokemon-card {
  border-radius: 16px !important;
  overflow: hidden;
  position: relative;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  transition: transform 0.2s ease-in-out, box-shadow 0.2s ease-in-out;
}

.pokemon-card:hover {
  transform: scale(1.05);
  box-shadow: 0 8px 20px rgba(0,0,0,0.2);
}

.pokemon-card-content {
  position: relative;
  height: 150px; /* Adjust height as needed */
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
}

.pokemon-card-background {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.pokeball-bg {
  position: absolute !important;
  bottom: -20px;
  right: -20px;
  width: 100px;
  height: 100px;
  opacity: 0.2;
  z-index: 2;
}

.pokemon-sprite {
  position: absolute !important;
  top: 5px;
  left: 50%;
  transform: translateX(-50%);
  width: 100px;
  height: 100px;
  z-index: 3;
}

.pokemon-info {
  position: relative;
  z-index: 4;
  padding: 8px;
  text-shadow: 1px 1px 3px rgba(0,0,0,0.4);
}
</style>