<script setup>
import axios from "axios";
import { ref } from "vue";

const pokemonData = ref(null);
const pokemonID = ref("");
const cargando = ref(false);
const error = ref("");
const debilidades = ref([]);

const TYPE_COLORS = {
  normal: "#A8A77A",
  fire: "#EE8130",
  water: "#6390F0",
  electric: "#F7D02C",
  grass: "#7AC74C",
  ice: "#96D9D6",
  fighting: "#C22E28",
  poison: "#A33EA1",
  ground: "#E2BF65",
  flying: "#A98FF3",
  psychic: "#F95587",
  bug: "#A6B91A",
  rock: "#B6A136",
  ghost: "#735797",
  dragon: "#6F35FC",
  dark: "#705746",
  steel: "#B7B7CE",
  fairy: "#F7B9E3",
};

function shadeColor(hex, percent) {
  const raw = (hex || "#dddddd").replace("#", "");
  const num = parseInt(raw, 16);
  let r = (num >> 16) + Math.round(2.55 * percent);
  let g = ((num >> 8) & 0x00ff) + Math.round(2.55 * percent);
  let b = (num & 0x0000ff) + Math.round(2.55 * percent);
  r = Math.max(0, Math.min(255, r));
  g = Math.max(0, Math.min(255, g));
  b = Math.max(0, Math.min(255, b));
  return "#" + ((r << 16) | (g << 8) | b).toString(16).padStart(6, "0");
}
function contrastText(hex) {
  const raw = (hex || "#ffffff").replace("#", "");
  const num = parseInt(raw, 16);
  const r = num >> 16,
    g = (num >> 8) & 0x00ff,
    b = num & 0x0000ff;
  const l = 0.299 * r + 0.587 * g + 0.114 * b;
  return l > 140 ? "#222" : "#fff";
}

async function buscarPokemon() {
  if (!pokemonID.value) {
    error.value = "Por favor ingrese un nombre o ID";
    return;
  }
  cargando.value = true;
  error.value = "";
  debilidades.value = [];
  pokemonData.value = null;
  try {
    const res = await axios.get(
      `https://pokeapi.co/api/v2/pokemon/${pokemonID.value.toLowerCase()}`
    );
    pokemonData.value = res.data;
    await obtenerDebilidades();
  } catch (e) {
    error.value =
      e.response?.status === 404
        ? "Pokémon no encontrado"
        : "Error al conectar con la API";
    pokemonData.value = null;
  } finally {
    cargando.value = false;
  }
}
async function obtenerDebilidades() {
  try {
    const promises = pokemonData.value.types.map((t) => axios.get(t.type.url));
    const responses = await Promise.all(promises);
    const all = responses.flatMap((r) =>
      r.data.damage_relations.double_damage_from.map((d) => d.name)
    );
    debilidades.value = Array.from(new Set(all));
  } catch (e) {
    console.error(e);
    error.value = "Error al obtener debilidades";
  }
}

function getTypeCols() {
  if (!pokemonData.value || !pokemonData.value.types)
    return ["#f3f3f3", "#f3f3f3"];
  const types = pokemonData.value.types.map((t) => t.type.name);
  const cols = types.map((t) => TYPE_COLORS[t] || "#ddd");
  if (cols.length === 1) return [cols[0], shadeColor(cols[0], -12)];
  return [cols[0], cols[1] || cols[0]];
}
function centerStyle() {
  const [c1, c2] = getTypeCols();
  return {
    background: `linear-gradient(180deg, ${c1} 0%, ${c2} 100%)`,
    color: contrastText(getTypeCols()[0]),
  };
}
function statFillColor() {
  return getTypeCols()[0] || "#d2b04a";
}
function weaknessColor(t) {
  return TYPE_COLORS[t] || "#ddd";
}
function enrichedStats() {
  if (!pokemonData.value) return [];
  return pokemonData.value.stats.map((s) => {
    const v = s.base_stat;
    const pct = Math.round((v / 255) * 100);
    let level = "low";
    if (v >= 170) level = "high";
    else if (v >= 85) level = "medium";
    return { name: s.stat.name.replace("-", " "), value: v, pct, level };
  });
}

function rnd(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}
async function buscarAleatorio() {
  pokemonID.value = String(rnd(1, 1010));
  await buscarPokemon();
}
</script>

<template>
  <div class="app">
    <!-- fondo principal: degradado global -->
    <div class="page-gradient"></div>

    <div class="ui">
      <header class="topbar">
        <div class="controls">
          <input
            v-model="pokemonID"
            @keyup.enter="buscarPokemon"
            placeholder="nombre o ID"
          />
          <button @click="buscarPokemon" :disabled="cargando">
            {{ cargando ? "Buscando..." : "Buscar" }}
          </button>
          <button class="rnd" @click="buscarAleatorio" :disabled="cargando">
            Aleatorio
          </button>
        </div>
        <div class="title">PokeFicha</div>
      </header>

      <main class="main-layout">
        <aside class="left-card" :style="centerStyle()">
          <div class="left-top">
            <div class="img-wrapper">
              <img
                :src="
                  pokemonData?.sprites?.other?.['official-artwork']
                    ?.front_default ||
                  pokemonData?.sprites?.front_default ||
                  pokemonData?.sprites?.front_shiny_female
                "
                :alt="pokemonData?.name || 'pokemon'"
                class="main-image"
                v-if="pokemonData"
                loading="lazy"
              />
              <div v-else class="placeholder">?</div>
            </div>
          </div>

          <div class="left-body">
            <h2 class="name">{{ pokemonData?.name || "---" }}</h2>
            <div class="id">#{{ pokemonData?.id || "--" }}</div>

            <div class="meta">
              <div class="meta-item">
                Alt:
                {{
                  pokemonData
                    ? (pokemonData.height / 10).toFixed(1) + " m"
                    : "--"
                }}
              </div>
              <div class="meta-item">
                Peso:
                {{
                  pokemonData
                    ? (pokemonData.weight / 10).toFixed(0) + " kg"
                    : "--"
                }}
              </div>
            </div>

            <div class="types-area">
              <h3>Tipos</h3>
              <div class="types">
                <span
                  v-for="t in pokemonData?.types || []"
                  :key="t.slot"
                  class="type"
                  :style="{
                    background: TYPE_COLORS[t.type.name] || '#eee',
                    color: contrastText(TYPE_COLORS[t.type.name] || '#eee'),
                  }"
                  >{{ t.type.name }}</span
                >
              </div>
            </div>

            <div class="weak-area">
              <h3>Debilidades</h3>
              <div class="weak-list">
                <span v-if="debilidades.length === 0" class="weak-empty"
                  >— —</span
                >
                <span
                  v-for="w in debilidades"
                  :key="w"
                  class="weak"
                  :style="{
                    background: weaknessColor(w),
                    color: contrastText(weaknessColor(w)),
                  }"
                  >{{ w }}</span
                >
              </div>
            </div>
          </div>

          <div
            class="left-border"
            :style="{
              borderColor: shadeColor(getTypeCols()[0] || '#ddd', -30),
            }"
          ></div>
        </aside>

        <section class="right-panel">
          <div class="stats-card">
            <h2>Estadísticas</h2>
            <div v-if="pokemonData" class="stats-list">
              <div v-for="s in enrichedStats()" :key="s.name" class="stat-row">
                <div class="stat-head">
                  <span class="sname">{{ s.name }}</span>
                  <span class="sval">{{ s.value }} / 255</span>
                </div>
                <div class="bar">
                  <div
                    class="fill"
                    :class="s.level"
                    :style="{
                      width: s.pct + '%',
                      background: `linear-gradient(90deg, ${statFillColor()} 0%, ${shadeColor(
                        statFillColor(),
                        20
                      )} 100%)`,
                    }"
                  ></div>
                </div>
              </div>
            </div>
            <div v-else class="no-p">Busca un Pokémon</div>
          </div>
        </section>
      </main>

    
    </div>
  </div>
</template>

<style scoped>
.app {
  position: relative;
  min-height: 100vh;
  font-family: Inter, system-ui, Arial;
  color: #222;
}

.page-gradient {
  position: fixed;
  inset: 0;
  z-index: 0;
  background: linear-gradient(135deg, #cdeffd 0%, #f7f4d8 45%, #ffd6d6 100%);
  filter: saturate(1.02);
}

.ui {
  position: relative;
  z-index: 2;
  max-width: 1200px;
  margin: 36px auto;
  padding: 20px;
  box-sizing: border-box;
}

.topbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  margin-bottom: 20px;
}
.title {
  font-weight: 800;
  font-size: 20px;
  color: rgba(0, 0, 0, 0.75);
}
.controls {
  display: flex;
  gap: 8px;
  align-items: center;
}
.controls input {
  padding: 8px 10px;
  border-radius: 10px;
  border: 1px solid rgba(0, 0, 0, 0.08);
  min-width: 160px;
}
.controls button {
  padding: 8px 12px;
  border-radius: 10px;
  border: 0;
  background: #2b6b4a;
  color: #fff;
  cursor: pointer;
  font-weight: 600;
}
.controls .rnd {
  background: #3a6fa6;
}

.main-layout {
  display: grid;
  grid-template-columns: 380px 1fr;
  gap: 22px;
  align-items: start;
}

.left-card {
  position: relative;
  border-radius: 18px;
  padding: 18px;
  box-shadow: 0 12px 30px rgba(18, 18, 18, 0.07);
  display: flex;
  flex-direction: column;
  gap: 12px;
  min-height: 520px;
  overflow: hidden;
}

.left-border {
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 8px;
  background: transparent;
  border-left: 8px solid #000;
  border-radius: 8px 0 0 8px;
  pointer-events: none;
}

.img-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 12px;
  background: rgba(255, 255, 255, 0.06);
  border-radius: 14px;
}
.main-image {
  width: 220px;
  height: 220px;
  object-fit: contain;
  filter: drop-shadow(0 18px 30px rgba(0, 0, 0, 0.14));
  transition: transform 0.28s;
}
.main-image:hover {
  transform: translateY(-8px);
}

.left-body {
  display: flex;
  flex-direction: column;
  gap: 8px;
  text-align: center;
  color: inherit;
}
.name {
  text-transform: capitalize;
  font-size: 22px;
  font-weight: 800;
  margin: 0;
}
.id {
  font-weight: 700;
  color: rgba(0, 0, 0, 0.12);
}
.meta {
  display: flex;
  justify-content: center;
  gap: 14px;
  margin-top: 6px;
  color: rgba(255, 255, 255, 0.95);
  font-weight: 700;
}
.meta-item {
  background: rgba(255, 255, 255, 0.08);
  padding: 6px 10px;
  border-radius: 8px;
  color: inherit;
}

.types-area,
.weak-area {
  margin-top: 10px;
  text-align: left;
  padding: 0 8px;
}
.types-area h3,
.weak-area h3 {
  margin: 0 0 8px 0;
  font-size: 14px;
  font-weight: 800;
  color: rgba(255, 255, 255, 0.95);
}
.types {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  justify-content: center;
}
.type {
  padding: 6px 10px;
  border-radius: 999px;
  font-weight: 700;
  box-shadow: 0 8px 18px rgba(0, 0, 0, 0.1);
  text-transform: uppercase;
  font-size: 13px;
}

.weak-list {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  justify-content: center;
}
.weak {
  padding: 6px 10px;
  border-radius: 8px;
  font-weight: 800;
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.08);
  text-transform: uppercase;
  font-size: 12px;
}
.weak-empty {
  color: rgba(255, 255, 255, 0.85);
}

.right-panel {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.stats-card {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 14px;
  padding: 18px;
  box-shadow: 0 12px 30px rgba(19, 19, 19, 0.06);
}
.stats-card h2 {
  margin: 0 0 12px 0;
  color: #222;
}
.stats-list {
  display: flex;
  flex-direction: column;
  gap: 14px;
}
.stat-row {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.stat-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.sname {
  text-transform: capitalize;
  font-weight: 700;
  color: #333;
}
.sval {
  color: #6b6b6b;
  font-weight: 700;
}
.bar {
  height: 18px;
  background: #fff;
  border-radius: 999px;
  border: 1px solid rgba(0, 0, 0, 0.06);
  overflow: hidden;
  box-shadow: inset 0 2px 6px rgba(0, 0, 0, 0.04);
}
.fill {
  height: 100%;
  transition: width 0.6s cubic-bezier(0.2, 0.8, 0.2, 1);
}

.extra-card {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 12px;
  padding: 12px;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.05);
}

.footer {
  margin-top: 18px;
  text-align: center;
  color: rgba(0, 0, 0, 0.45);
}

@media (max-width: 980px) {
  .main-layout {
    grid-template-columns: 1fr;
  }
  .left-card {
    min-height: 420px;
  }
}
</style>
