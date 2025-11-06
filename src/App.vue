<script setup>
import axios from "axios";
import { ref } from "vue";

const pokemonData = ref(null);
const pokemonID = ref("");
const cargando = ref(false);
const error = ref("");
const debilidades = ref([]);



const tiposColor ={

  normal: '#A8A878',
  fire: '#F08030',
  water: '#6890F0',
  electric: '#F8D030',
  grass: '#78C850',
  ice: '#98D8D8',
  fighting: '#C03028',
  poison: '#A040A0',
  ground: '#E0C068',
  flying: '#A890F0',
  psychic: '#F85888',
  bug: '#A8B820',
  rock: '#B8A038',
  ghost: '#705898',
  dragon: '#7038F8',
  dark: '#705848',
  steel: '#B8B8D0',
  fairy: '#EE99AC'
};


const getGradianteColor = (types) =>{
  if(!types || types.length ===0) return  'linear-gradient(to right , #666666,#888888)';


  if(types.length === 1){
    const color = tiposColor[types[0].type.name] || '#666666';
    return `linear-gradient(to right, ${color}, ${adjustColor(color, -20)})`;
  }

  const color1= tiposColor[types[0].type.name] || '#666666';
  const color2 = tiposColor[types[1].type.name] || '#888888';
  return `linear-gradient(to right, ${color1} ,${color2})`;
}



 const adjustColor= (color,amount) =>{
   const hex = color.replace('#', '');
  const r = Math.max(Math.min(parseInt(hex.substring(0,2), 16) + amount, 255), 0);
  const g = Math.max(Math.min(parseInt(hex.substring(2,4), 16) + amount, 255), 0);
  const b = Math.max(Math.min(parseInt(hex.substring(4,6), 16) + amount, 255), 0);

  return `#${r.toString(16).padStart(2,'0')}${g.toString(16).padStart(2,'0')}${b.toString(16).padStart(2,'0')}`;
 };















const buscarPokemon = async () => {
  if (!pokemonID.value) {
    error.value = "Por favor ingrese  un nombre o ID";

    return;
  }

  cargando.value = true;
  error.value = "";
  debilidades.value = [];

  try {
    const response = await axios.get(
      `https://pokeapi.co/api/v2/pokemon/${pokemonID.value.toLowerCase()}`
    );

    pokemonData.value = response.data;

    await obtenerDebilidades();

    console.log(pokemonData);
  } catch (err) {
    error.value =
      err.response?.status === 404
        ? "pokemon no encontrado"
        : "error al conectar con la API";
    pokemonData.value = null;
  } finally {
    cargando.value = false;
  }
};

const obtenerDebilidades = async () => {
  try {
    const tiposPromesas = pokemonData.value.types.map((type) =>
      axios.get(type.type.url)
    );

    const tiposResponses = await Promise.all(tiposPromesas);

    const todasDebilidades = tiposResponses.flatMap((response) =>
      response.data.damage_relations.double_damage_from.map((d) => d.name)
    );

    debilidades.value = [...new Set(todasDebilidades)];
  } catch (err) {
    console.error("error al obtener debilidades", err);
    error.value = "error al obtener debilidades";
  }
};



function rnd(min,max){
  return Math.floor(Math.random() *(max-min+1))+ min;
}

async function buscarAleatorio(){
  pokemonID.value = String(rnd(1,1010));
  await buscarPokemon();
}


</script>

<template>
  <div class="container">
    <header>
      <div class="lupa">
        <label for="pokemonInput">
          <input v-model="pokemonID" type="text" id="pokemonInput" />
        </label>
        <button class="boton" type="button" @click="buscarPokemon">
          Buscar
        </button>
        <button class="boton-aleatorio" type="button" @click="buscarAleatorio" :disabled="cargando">
          aleatorio
        </button>
      </div>
    </header>

    <main class="main" v-if="pokemonData">
      <div class="content-wrapper">
        <!-- Columna izquierda -->
        <div class="left-column" 
          :style="{ 
            '--pokemon-gradient': getGradianteColor(pokemonData.types),
            background: 'var(--pokemon-gradient)',
            transition: 'background 0.3s ease'
          }"
        >
          <section class="Pokemon-presentacion">
            <div class="img-wrapper">
              <img
                :src="pokemonData?.sprites?.other?.['official-artwork']?.front_default"
                :alt="pokemonData?.name || 'pokemon'"
                class="main-image"
                v-if="pokemonData"
                loading="lazy"
              />
              <div v-else class="placeholder">?</div>
            </div>
            <h2>{{ pokemonData.name }}</h2>
          </section>

          <div class="informacion-pokemon">
            <h2>ID:{{ pokemonData.id }}</h2>
            <div class="peso-altura">
              <h3>peso :{{ pokemonData.weight / 10 }} kg</h3>
              <h3>altura :{{ pokemonData.height / 10 }} m</h3>
            </div>

            <div class="tipos">
              <h2>Tipos:</h2>
              <ul>
                <li v-for="type in pokemonData.types" :key="type.slot" 
                :style="{ backgroundColor: tiposColor[type.type.name] }">
                  {{ type.type.name }}
                </li>
              </ul>
            </div>

            <div class="debilidades" v-if="debilidades.length > 0">
              <h2>Debilidades:</h2>
              <ul>
                <li v-for="debilidad in debilidades" :key="debilidad" 
                 :style="{ backgroundColor: tiposColor[debilidad] }">
                  {{ debilidad }}
                </li>
              </ul>
            </div>
          </div>
        </div>

        <!-- Columna derecha - Estadísticas -->
        <div class="right-column">
          <div class="estadisticas">
            <h2>Estadísticas:</h2>
            <div class="stats-container">
              <div
                v-for="stat in pokemonData.stats"
                :key="stat.stat.name"
                class="stat-item"
              >
                <div class="stat-header">
                  <span class="stat-name">{{ stat.stat.name }}:</span>
                  <span class="stat-value">{{ stat.base_stat }}</span>
                </div>
                <div class="stat-bar">
                  <div
                    class="stat-fill"
                    :style="{ 
                      width: `${(stat.base_stat / 255) * 100}%`,
                      background: getGradianteColor(pokemonData.types)
                    }"
                  ></div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>

.container {
  max-width: 1000px; 
  margin: 0 auto;
  padding: 1rem;
}
.content-wrapper{
  display: flex;
  gap:1rem;
  align-items: flex-start;
  justify-content: center;
  margin: 0 auto;
  max-width: 1200px;
}




.right-column{
  width: 450px;
  flex-shrink: 0;
  height: 50px;
}


.estadisticas{
    background: rgba(255, 255, 255, 0.95);
  border-radius: 14px;
  padding: 12px;
  box-shadow: 0 12px 30px rgba(19, 19, 19, 0.06);
}


.stats-container{
  display: flex;
  flex-direction: column;
  gap: 1rem;
    background: rgba(255, 255, 255, 0.95);
}

.stat-item{
  margin: 0.5rem 0;
}


.stat-header{
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.3rem;

}

.stat-name{
  display: inline-block;
  width: 120px;
  margin-right: 1rem;
  font-size: 0.9rem;
  color: rgba(6, 6, 6, 0.9);
}


.stat-value{
  font-weight: bold;
  min-width: 40px;
  text-align: right;
}

.stat-bar{
  background: rgba(255, 255, 255, 0.2);

  height: 10px;
  border-radius: 5px;
  margin-top: 0.5rem;
  overflow: hidden;
  box-shadow: inset 0  2px 4px  rgba(0, 0, 0, 0.1)
}


.stat-fill{
   height: 100%;
  transition: width 0.3s ease, background 0.3s ease;
  border-radius: 5px;
  background-size: 200% 100% !important;
}

h2, h3, .stat-name, .stat-value {
  text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.3);
  margin: 0.3rem 0;
}

.img-wrapper{
   background: rgba(255, 255, 255, 0.1); 
  border-radius: 20px;
  padding: 1.5rem;
  backdrop-filter: blur(8px); 
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  margin: 1rem;
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.main-image {
  width: 230px;
  height: 150px;
  object-fit: contain;
  filter: drop-shadow(0 10px 15px rgba(0, 0, 0, 0.1));
  transition: all 0.5s cubic-bezier(0.4, 0, 0.2, 1);
  display: block;
  margin: 0 auto;
  padding: 1rem;
}

.main-image:hover {
  transform: translateY(-10px);
  filter: drop-shadow(0 20px 25px rgba(0, 0, 0, 0.2));
}


.tipos,
.debilidades {
  padding: 0.3rem; 
}
.tipos ul, 
.debilidades ul{
  display: grid;
  grid-template-columns: repeat(3, 1fr); 
  gap: 0.5rem;
  padding: 0;
  list-style: none;
}

.tipos li,.debilidades li{
   display: flex;
  justify-content: center;
  align-items: center;
  
  border-radius: 10px;
  color: white;
  text-shadow:  1px 1px 2px rgba(0, 0, 0, 0.3);
  font-weight: bold;
  text-transform: capitalize;
  transition: all 0.3s ease;
   box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
    padding: 0.3rem 0.5rem;
  font-size: 0.8rem; 
  min-height: 24px; 
}
.tipos li:hover,
.debilidades li:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}



.left-column {
  display: flex;
  flex-direction: column;
  width: 380px; 
  flex-shrink: 0;
  border-radius: 14px;
  overflow: hidden;
  height: 650px;
  gap: 1px;
}



.Pokemon-presentacion h2{
  text-align: center;
  font-size: 1rem;
  margin: 1rem 0;
  color: white;

}

.informacion-pokemon h2{
  text-align: center;
  font-size: 1rem;
}
.peso-altura{
  display: flex;
  flex-wrap: wrap;
  font-size: 0.7rem;
   justify-content: center;
  gap: 1rem;
  padding: 0.8rem;
 
 
  margin: 0.8rem 0;
}

.peso-altura h3 {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.9rem;
  color: white;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
  padding: 0.5rem 1rem;
  background: rgba(255, 255, 255, 0.15);
  border-radius: 8px;
  margin: 0;
  min-width: 120px;
  justify-content: center;
 
}


.lupa {
     display: flex;
  justify-content: center;
  align-items: center;
  gap: 0.5rem;
  padding: 0.8rem;
  margin: 0.5rem auto;
  max-width: 450px;
 
  background: transparent;
  backdrop-filter: blur(0); 
  box-shadow: none; 
}
#pokemonInput {
 padding: 0.6rem 1rem;
  border: 2px solid rgba(255, 255, 255, 0.2);
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.9);
  font-size: 1rem;
  width: 160px;
  transition: all 0.3s ease;
  outline: none;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}
.boton, .boton-aleatorio {
  padding: 0.8rem 1.5rem;
  border: none;
  border-radius: 12px;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.boton {
  background: #ed6300;
  color: white;
  box-shadow: 0 4px 8px rgba(76, 175, 80, 0.3);
}
.boton-aleatorio {
  background: #2196F3;
  color: white;
  box-shadow: 0 4px 8px rgba(76, 0, 198, 0.3);
}
.boton:hover {
  background: #9bc200;
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(76, 175, 80, 0.4);
}
.boton-aleatorio:hover {
  background: #c1e51e;
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(33, 150, 243, 0.4);
}





@media (max-width: 768px) {
  .content-wrapper {
    flex-direction: column;
    align-items: center;
    gap: 1rem;
  }

  .left-column,
  .right-column {
    width: 100%;
    max-width: 380px;
    height: auto;
  }

  .main-image {
    width: 180px;
    height: 180px;
  }

  .img-wrapper {
    padding: 0.8rem;
    margin: 0.5rem;
  }
}

@media (max-width: 480px) {
  .container {
    padding: 0.5rem;
  }

  .lupa {
    flex-direction: column;
    gap: 0.5rem;
  }

  #pokemonInput,
  .boton,
  .boton-aleatorio {
    width: 100%;
    max-width: 250px;
    padding: 0.6rem 1rem;
    font-size: 0.8rem;
  }

  .left-column,
  .right-column {
    max-width: 300px;
  }

  .main-image {
    width: 150px;
    height: 150px;
  }

  .Pokemon-presentacion h2 {
    font-size: 0.9rem;
  }

  .peso-altura h3 {
    min-width: 100px;
    font-size: 0.8rem;
    padding: 0.3rem 0.8rem;
  }

  .tipos ul,
  .debilidades ul {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 350px) {
  .left-column,
  .right-column {
    max-width: 280px;
  }

  .main-image {
    width: 130px;
    height: 130px;
  }

  .stat-name {
    width: 90px;
    font-size: 0.75rem;
  }

  .stat-value {
    font-size: 0.75rem;
  }

  .tipos li,
  .debilidades li {
    font-size: 0.7rem;
    padding: 0.2rem 0.4rem;
    min-height: 20px;
  }

  .peso-altura {
    gap: 0.5rem;
  }

  .peso-altura h3 {
    min-width: 90px;
    font-size: 0.7rem;
  }
}


.content-wrapper {
  min-height: 0; 
}

.left-column {
  height: auto; 
}

.right-column {
  height: auto; 
}


.estadisticas {
  height: auto;
  margin-bottom: 1rem;
}









</style>
