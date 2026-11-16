<script setup>
import { computed, onMounted, ref } from "vue";

const API = "https://pokeapi.co/api/v2";
const PAGE_SIZE = 24;

const pokemon = ref([]);
const query = ref("");
const page = ref(1);
const loading = ref(true);
const error = ref("");
const selected = ref(null);
const detailLoading = ref(false);

const filtered = computed(() => {
  const term = query.value.trim().toLowerCase();
  return pokemon.value.filter((item) => item.name.includes(term));
});

const pageCount = computed(() =>
  Math.max(1, Math.ceil(filtered.value.length / PAGE_SIZE)),
);

const visiblePokemon = computed(() => {
  const start = (page.value - 1) * PAGE_SIZE;
  return filtered.value.slice(start, start + PAGE_SIZE);
});

function pokemonId(url) {
  return url.match(/\/(\d+)\/?$/)?.[1] ?? "";
}

function spriteUrl(item) {
  const id = pokemonId(item.url);
  return `https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/other/official-artwork/${id}.png`;
}

function formatName(name = "") {
  return name.replaceAll("-", " ");
}

async function loadPokemon() {
  loading.value = true;
  error.value = "";

  try {
    const response = await fetch(`${API}/pokemon?limit=2000`);
    if (!response.ok) throw new Error("Could not load Pokémon.");

    const data = await response.json();
    pokemon.value = data.results;
  } catch {
    error.value = "Pokémon couldn’t load. Check your connection and try again.";
  } finally {
    loading.value = false;
  }
}

function updateSearch() {
  page.value = 1;
}

async function openPokemon(item) {
  selected.value = null;
  detailLoading.value = true;

  try {
    const response = await fetch(item.url);
    if (!response.ok) throw new Error("Could not load Pokémon details.");
    selected.value = await response.json();
  } catch {
    error.value = "Couldn’t load those Pokémon details. Please try again.";
  } finally {
    detailLoading.value = false;
  }
}

onMounted(loadPokemon);
</script>

<template>
  <main class="app-shell">
    <header class="topbar">
      <a class="brand" href="#" aria-label="Pokédex home">
        <span class="brand-ball" aria-hidden="true"><i></i></span>
        <span>Poke<span class="brand-accent">dex</span></span>
      </a>
      <span class="topbar-note">The Pokémon encyclopedia</span>
    </header>

    <section class="hero">
      <div class="hero-copy">
        <p class="eyebrow"><span class="eyebrow-dot"></span> YOUR FIELD GUIDE</p>
        <h1>Find your next<br /><span>favorite Pokémon.</span></h1>
        <p class="hero-description">
          Explore stats, types, and details from across the Pokémon world.
        </p>
      </div>
      <div class="hero-art" aria-hidden="true">
        <span class="orbit orbit-one"></span>
        <span class="orbit orbit-two"></span>
        <span class="hero-ball"><i></i></span>
        <span class="spark spark-one">✦</span>
        <span class="spark spark-two">✧</span>
      </div>
    </section>

    <section class="catalog" aria-label="Pokémon catalog">
      <div class="catalog-heading">
        <div>
          <p class="eyebrow">THE COLLECTION</p>
          <h2>Pokémon <span class="result-count">{{ filtered.length }}</span></h2>
        </div>

        <label class="search">
          <span class="search-icon" aria-hidden="true">⌕</span>
          <input
            v-model="query"
            @input="updateSearch"
            type="search"
            placeholder="Search Pokémon..."
            aria-label="Search Pokémon"
          />
          <kbd>/</kbd>
        </label>
      </div>

      <p v-if="error" class="message error-message" role="alert">{{ error }}</p>
      <p v-else-if="loading" class="message">Loading Pokémon…</p>
      <p v-else-if="filtered.length === 0" class="message">
        No Pokémon found for “{{ query }}”.
      </p>

      <div v-else class="pokemon-grid">
        <button
          v-for="item in visiblePokemon"
          :key="item.name"
          class="pokemon-card"
          type="button"
          @click="openPokemon(item)"
        >
          <span class="card-number">#{{ pokemonId(item.url).padStart(4, "0") }}</span>
          <span class="card-art">
             <img :src="spriteUrl(item)" :alt="formatName(item.name)" loading="lazy" />
          </span>
          <span class="card-name">{{ formatName(item.name) }}</span>
          <span class="card-link">View details <span aria-hidden="true">↗</span></span>
        </button>
      </div>

      <nav v-if="!loading && filtered.length > PAGE_SIZE" class="pagination" aria-label="Pagination">
        <button
          class="page-button"
          type="button"
          :disabled="page === 1"
          @click="page--"
        >
          ← Previous
        </button>
        <span class="page-indicator">Page {{ page }} of {{ pageCount }}</span>
        <button
          class="page-button"
          type="button"
          :disabled="page >= pageCount"
          @click="page++"
        >
          Next →
        </button>
      </nav>
    </section>

    <div
      v-if="selected || detailLoading"
      class="modal-backdrop"
      @click.self="selected = null"
      @keydown.esc="selected = null"
    >
      <section
        v-if="selected"
        class="detail-modal"
        role="dialog"
        aria-modal="true"
        :aria-label="`${formatName(selected.name)} details`"
      >
        <button class="close-button" type="button" aria-label="Close" @click="selected = null">
          ×
        </button>

        <div class="detail-art">
          <span class="detail-number">#{{ String(selected.id).padStart(4, "0") }}</span>
          <img
            :src="selected.sprites.other['official-artwork'].front_default || selected.sprites.front_default"
            :alt="formatName(selected.name)"
          />
        </div>

        <div class="detail-content">
          <p class="eyebrow">POKÉMON PROFILE</p>
          <h2>{{ formatName(selected.name) }}</h2>

          <div class="type-list">
            <span
              v-for="entry in selected.types"
              :key="entry.type.name"
              class="type-chip"
              :class="`type-${entry.type.name}`"
            >
              {{ entry.type.name }}
            </span>
          </div>

          <div class="measurements">
            <div><span>Height</span><strong>{{ (selected.height / 10).toFixed(1) }} m</strong></div>
            <div><span>Weight</span><strong>{{ (selected.weight / 10).toFixed(1) }} kg</strong></div>
            <div><span>Base EXP</span><strong>{{ selected.base_experience ?? "—" }}</strong></div>
          </div>

          <h3>Base stats</h3>
          <div v-for="stat in selected.stats" :key="stat.stat.name" class="stat-row">
            <span class="stat-name">{{ stat.stat.name.replace("-", " ") }}</span>
            <div class="stat-track">
              <span :style="{ width: `${Math.min(stat.base_stat, 150) / 150 * 100}%` }"></span>
            </div>
            <strong>{{ stat.base_stat }}</strong>
          </div>
        </div>
      </section>

      <section v-else class="detail-modal loading-modal" role="status">
        Loading Pokémon details…
      </section>
    </div>

    <footer class="footer">
      <span>Made for curious trainers.</span>
      <span>Pokémon data provided by PokéAPI.</span>
    </footer>
  </main>
</template>

<style scoped>
.app-shell {
  width: min(1120px, calc(100% - 48px));
  margin: 0 auto;
}

.topbar {
  height: 76px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid #e9eaf0;
}

.brand {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  color: #202336;
  text-decoration: none;
  font-size: 20px;
  font-weight: 850;
  letter-spacing: -0.8px;
}

.brand-accent {
  color: #ea5548;
}

.brand-ball,
.hero-ball {
  position: relative;
  display: inline-block;
  overflow: hidden;
  width: 27px;
  height: 27px;
  border: 2px solid #25273a;
  border-radius: 50%;
  background: linear-gradient(to bottom, #ed5b4f 0 47%, #25273a 47% 54%, #fff 54%);
}

.brand-ball i,
.hero-ball i {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 9px;
  height: 9px;
  transform: translate(-50%, -50%);
  border: 2px solid #25273a;
  border-radius: 50%;
  background: #fff;
}

.topbar-note {
  color: #898b9b;
  font-size: 13px;
}

.hero {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: space-between;
  min-height: 310px;
  padding: 46px 72px;
  margin: 32px 0 52px;
  overflow: hidden;
  border: 1px solid #f0d4ce;
  border-radius: 24px;
  background: linear-gradient(110deg, #fff7f3, #fffaf0 72%, #fff3ee);
}

.hero-copy {
  position: relative;
  z-index: 1;
}

.eyebrow {
  margin: 0 0 12px;
  color: #e65b4d;
  font-size: 11px;
  font-weight: 800;
  letter-spacing: 1.7px;
}

.eyebrow-dot {
  display: inline-block;
  width: 7px;
  height: 7px;
  margin-right: 6px;
  border-radius: 50%;
  background: #ed6556;
}

.hero h1 {
  margin: 0;
  color: #202336;
  font-size: clamp(38px, 5vw, 56px);
  line-height: 1.06;
  letter-spacing: -2.8px;
}

.hero h1 span {
  color: #e65b4d;
}

.hero-description {
  margin: 18px 0 0;
  color: #77798a;
  font-size: 15px;
}

.hero-art {
  position: absolute;
  right: 9%;
  width: 245px;
  height: 245px;
}

.orbit {
  position: absolute;
  inset: 18px;
  border: 1px solid #efc8bb;
  border-radius: 50%;
}

.orbit-two {
  inset: 0;
  border-style: dashed;
  opacity: 0.7;
}

.hero-ball {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 112px;
  height: 112px;
  border-width: 7px;
  transform: translate(-50%, -50%) rotate(-18deg);
  box-shadow: 0 16px 40px #d99c8855;
}

.hero-ball i {
  width: 35px;
  height: 35px;
  border-width: 6px;
}

.spark {
  position: absolute;
  color: #e65b4d;
  font-size: 27px;
}

.spark-one {
  top: 4px;
  right: 32px;
}

.spark-two {
  bottom: 8px;
  left: 7px;
  color: #e6a543;
}

.catalog-heading {
  display: flex;
  align-items: end;
  justify-content: space-between;
  gap: 24px;
  margin-bottom: 25px;
}

.catalog-heading .eyebrow {
  margin-bottom: 8px;
}

.catalog h2 {
  margin: 0;
  font-size: 27px;
  letter-spacing: -1px;
}

.result-count {
  margin-left: 6px;
  color: #a0a2b0;
  font-size: 15px;
  font-weight: 500;
  letter-spacing: 0;
}

.search {
  display: flex;
  align-items: center;
  gap: 10px;
  width: min(330px, 100%);
  height: 46px;
  padding: 0 13px;
  border: 1px solid #e6e7ed;
  border-radius: 12px;
  background: #fff;
  color: #8c8e9d;
}

.search:focus-within {
  border-color: #e7968d;
  box-shadow: 0 0 0 3px #e65b4d14;
}

.search-icon {
  font-size: 25px;
  line-height: 1;
  transform: rotate(-20deg);
}

.search input {
  width: 100%;
  min-width: 0;
  border: 0;
  outline: 0;
  color: #202336;
  background: transparent;
  font-size: 13px;
}

.search kbd {
  padding: 2px 6px;
  border: 1px solid #e9eaf0;
  border-radius: 5px;
  background: #fafafd;
  font-size: 11px;
}

.pokemon-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 16px;
}

.pokemon-card {
  position: relative;
  display: flex;
  min-width: 0;
  flex-direction: column;
  align-items: center;
  padding: 13px 16px 16px;
  border: 1px solid #eaebf0;
  border-radius: 16px;
  background: #fff;
  color: inherit;
  text-align: left;
  cursor: pointer;
  transition: transform 160ms ease, box-shadow 160ms ease, border-color 160ms ease;
}

.pokemon-card:hover {
  transform: translateY(-4px);
  border-color: #f0c2bb;
  box-shadow: 0 12px 28px #31334a10;
}

.card-number {
  align-self: flex-end;
  color: #a1a3b0;
  font-size: 11px;
  font-weight: 700;
}

.card-art {
  display: grid;
  width: 100%;
  height: 150px;
  place-items: center;
  margin: 4px 0 8px;
  border-radius: 12px;
  background: #f7f8fc;
}

.card-art img {
  width: 128px;
  height: 128px;
  object-fit: contain;
  transition: transform 180ms ease;
}

.pokemon-card:hover .card-art img {
  transform: scale(1.07);
}

.card-name {
  width: 100%;
  overflow: hidden;
  color: #27293c;
  font-size: 15px;
  font-weight: 750;
  text-align: center;
  text-overflow: ellipsis;
  text-transform: capitalize;
  white-space: nowrap;
}

.card-link {
  margin-top: 7px;
  color: #a0a2af;
  font-size: 11px;
}

.message {
  padding: 40px 0;
  color: #77798a;
  text-align: center;
}

.error-message {
  color: #b63b34;
}

.pagination {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 20px;
  padding: 32px 0 10px;
}

.page-button {
  padding: 10px 14px;
  border: 1px solid #e6e7ed;
  border-radius: 9px;
  background: #fff;
  color: #35374a;
  cursor: pointer;
}

.page-button:disabled {
  color: #b9bbc5;
  cursor: not-allowed;
}

.page-indicator {
  color: #77798a;
  font-size: 13px;
}

.footer {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  margin-top: 56px;
  padding: 20px 0 26px;
  border-top: 1px solid #e9eaf0;
  color: #999baa;
  font-size: 12px;
}

.modal-backdrop {
  position: fixed;
  z-index: 5;
  inset: 0;
  display: grid;
  padding: 20px;
  place-items: center;
  background: #20233699;
  backdrop-filter: blur(4px);
}

.detail-modal {
  position: relative;
  display: grid;
  grid-template-columns: 0.9fr 1.1fr;
  width: min(700px, 100%);
  max-height: min(760px, 90vh);
  overflow: auto;
  border-radius: 22px;
  background: #fff;
  box-shadow: 0 24px 80px #17182b40;
}

.close-button {
  position: absolute;
  z-index: 1;
  top: 14px;
  right: 16px;
  width: 34px;
  height: 34px;
  border: 0;
  border-radius: 50%;
  background: #fff;
  color: #434559;
  font-size: 24px;
  cursor: pointer;
}

.detail-art {
  position: relative;
  display: grid;
  min-height: 360px;
  padding: 28px;
  place-items: center;
  background: #fff4ef;
}

.detail-art img {
  width: min(260px, 100%);
  aspect-ratio: 1;
  object-fit: contain;
}

.detail-number {
  position: absolute;
  top: 20px;
  left: 22px;
  color: #c99e94;
  font-size: 13px;
  font-weight: 750;
}

.detail-content {
  padding: 42px 30px 30px;
}

.detail-content h2 {
  margin: 0 0 12px;
  font-size: 32px;
  letter-spacing: -1.4px;
  text-transform: capitalize;
}

.type-list {
  display: flex;
  gap: 8px;
  margin-bottom: 24px;
}

.type-chip {
  padding: 6px 11px;
  border-radius: 99px;
  background: #eef0f4;
  color: #535568;
  font-size: 12px;
  font-weight: 700;
  text-transform: capitalize;
}

.type-fire { background: #ffebe1; color: #c75939; }
.type-water { background: #e3f1ff; color: #3979b9; }
.type-grass { background: #e8f6e8; color: #4d914f; }
.type-electric { background: #fff5d8; color: #a77d12; }
.type-psychic { background: #ffe8ef; color: #b84768; }
.type-ice { background: #e3f8f8; color: #388c91; }
.type-dragon { background: #eee9ff; color: #6650b4; }
.type-dark { background: #e9e6e5; color: #5d4a46; }
.type-fairy { background: #ffeaf6; color: #ae5c8f; }
.type-fighting { background: #ffebe7; color: #b54d3b; }
.type-poison { background: #f3e6f8; color: #8b53a1; }
.type-ground { background: #f7eedb; color: #96753f; }
.type-rock { background: #f1eedf; color: #817644; }
.type-bug { background: #eff5d9; color: #718333; }
.type-ghost { background: #ece9fa; color: #65598e; }
.type-steel { background: #e9eef2; color: #667988; }
.type-flying { background: #edeaff; color: #746ab1; }
.type-normal { background: #f0f0ed; color: #77776f; }

.measurements {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
  margin-bottom: 24px;
}

.measurements div {
  display: grid;
  gap: 6px;
  padding: 11px 9px;
  border-radius: 10px;
  background: #f7f8fc;
}

.measurements span {
  color: #9294a2;
  font-size: 10px;
}

.measurements strong {
  color: #34364a;
  font-size: 12px;
}

.detail-content h3 {
  margin: 0 0 12px;
  font-size: 14px;
}

.stat-row {
  display: grid;
  grid-template-columns: 90px 1fr 28px;
  align-items: center;
  gap: 9px;
  margin: 9px 0;
  font-size: 10px;
}

.stat-name {
  color: #737586;
  text-transform: capitalize;
}

.stat-row strong {
  color: #45475a;
  text-align: right;
}

.stat-track {
  height: 6px;
  overflow: hidden;
  border-radius: 99px;
  background: #eeeef3;
}

.stat-track span {
  display: block;
  height: 100%;
  border-radius: inherit;
  background: #e96b5c;
}

.loading-modal {
  display: grid;
  min-height: 150px;
  place-items: center;
  color: #77798a;
}

@media (max-width: 760px) {
  .hero {
    min-height: 290px;
    padding: 36px;
  }

  .hero-art {
    right: -75px;
    opacity: 0.48;
  }

  .pokemon-grid {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }

  .detail-modal {
    grid-template-columns: 1fr;
  }

  .detail-art {
    min-height: 240px;
  }

  .detail-art img {
    width: 190px;
  }
}

@media (max-width: 520px) {
  .app-shell {
    width: min(100% - 28px, 1120px);
  }

  .topbar-note {
    font-size: 11px;
  }

  .hero {
    min-height: 270px;
    padding: 28px 22px;
    margin: 20px 0 38px;
  }

  .hero h1 {
    font-size: 40px;
  }

  .hero-description {
    max-width: 250px;
    line-height: 1.5;
  }

  .catalog-heading {
    align-items: stretch;
    flex-direction: column;
  }

  .search {
    width: 100%;
  }

  .pokemon-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 11px;
  }

  .card-art {
    height: 125px;
  }

  .card-art img {
    width: 110px;
    height: 110px;
  }

  .footer {
    flex-direction: column;
    margin-top: 38px;
  }

  .pagination {
    gap: 10px;
  }

  .page-button {
    padding: 9px;
    font-size: 12px;
  }
}

</style>
