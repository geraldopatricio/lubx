<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import { Bar } from 'vue-chartjs';
import { 
  Chart as ChartJS, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale 
} from 'chart.js';
import { Car, Bike, Search, Eraser } from 'lucide-vue-next';
import mapboxgl from 'mapbox-gl';
import 'mapbox-gl/dist/mapbox-gl.css';

// Registrar Chart.js
ChartJS.register(CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend);

// --- MAPBOX ---
const mapContainer = ref(null);
const map = ref(null);
const MAPBOX_TOKEN = import.meta.env.VITE_MAPBOX_TOKEN || 'SUA_CHAVE_AQUI';

onMounted(() => {
  if (!MAPBOX_TOKEN) return;
  mapboxgl.accessToken = MAPBOX_TOKEN;
  map.value = new mapboxgl.Map({
    container: mapContainer.value,
    style: 'mapbox://styles/mapbox/light-v11',
    center: [-55.0, -14.2],
    zoom: 3,
    attributionControl: false
  });
  map.value.addControl(new mapboxgl.NavigationControl(), 'top-right');
});

onUnmounted(() => {
  if (map.value) map.value.remove();
});

// --- ESTADOS ---
const veiculoTipo = ref('carro');
const anoInicio = ref(2000);
const anoFim = ref(2025);
const marcaSearch = ref('');

// Lista de Marcas (Baseado no Print)
const marcas = ref([
  { name: 'ACURA', checked: false },
  { name: 'ALFA ROMEO', checked: false },
  { name: 'ASIA', checked: false },
  { name: 'ASTON MARTIN', checked: false },
  { name: 'AUDI', checked: false },
  { name: 'BENTLEY', checked: false },
  { name: 'BMW', checked: false },
  { name: 'BYD', checked: false },
  { name: 'CADILLAC', checked: false },
  { name: 'CAOA CHERY', checked: false },
  { name: 'CITROEN', checked: false },
  { name: 'DODGE', checked: false },
  { name: 'FERRARI', checked: false },
  { name: 'FIAT', checked: false },
  { name: 'FORD', checked: false },
  { name: 'GM - CHEVROLET', checked: false },
  { name: 'HONDA', checked: false },
  { name: 'HYUNDAI', checked: false },
  { name: 'JEEP', checked: false },
]);

// Filtro de Marcas
const filteredMarcas = computed(() => {
  if (!marcaSearch.value) return marcas.value;
  return marcas.value.filter(m => m.name.toLowerCase().includes(marcaSearch.value.toLowerCase()));
});

// --- DADOS GRÁFICOS (Mantendo padrão) ---
const cityChartData = {
  labels: ['São Paulo', 'Belo Horizonte', 'Rio de Janeiro', 'Brasília', 'Curitiba', 'Goiânia', 'Salvador'],
  datasets: [{ label: 'Capacidade', backgroundColor: '#f97316', borderRadius: 4, data: [65, 45, 40, 30, 25, 15, 10] }]
};

const viscosityChartData = {
  labels: ['5W30', '5W40', '15W40', '0W20', '10W40', '10W30', '0W30'],
  datasets: [{ label: 'Volume', backgroundColor: '#f97316', borderRadius: 4, data: [80, 55, 40, 25, 20, 10, 5] }]
};

const chartOptions = {
  indexAxis: 'y', responsive: true, maintainAspectRatio: false,
  plugins: { legend: { display: false } },
  scales: { x: { display: false }, y: { grid: { display: false } } }
};

// --- DADOS TABELA ---
const tableData = [
  { marca: 'BYD', anoModelo: '2025 YUAN PRO GS', lubrificante: '-', frota: 127, capacidade: '-', litros: '-' },
  { marca: 'BYD', anoModelo: '2025 YUAN PLUS GL', lubrificante: '-', frota: 30, capacidade: '-', litros: '-' },
  { marca: 'VOLVO', anoModelo: '2025 XC90 T8 AWD', lubrificante: '0W20 C5', frota: 48, capacidade: 5.60, litros: 268.80 },
  { marca: 'VOLVO', anoModelo: '2025 XC90 T8 ULT', lubrificante: '0W20 C5', frota: 128, capacidade: 5.60, litros: 716.80 },
  { marca: 'VOLVO', anoModelo: '2025 XC90 T8 PLUS', lubrificante: '0W20 C5', frota: 46, capacidade: 5.60, litros: 257.60 },
  { marca: 'VOLVO', anoModelo: '2025 XC90 T8 AWD', lubrificante: '0W20 C5', frota: 48, capacidade: 5.60, litros: 268.80 },
  { marca: 'VOLVO', anoModelo: '2025 XC90 T8 ULT', lubrificante: '0W20 C5', frota: 128, capacidade: 5.60, litros: 716.80 },
  { marca: 'VOLVO', anoModelo: '2025 XC90 T8 PLUS', lubrificante: '0W20 C5', frota: 46, capacidade: 5.60, litros: 257.60 },
  { marca: 'VOLVO', anoModelo: '2025 XC90 T8 AWD', lubrificante: '0W20 C5', frota: 48, capacidade: 5.60, litros: 268.80 },
  { marca: 'VOLVO', anoModelo: '2025 XC90 T8 ULT', lubrificante: '0W20 C5', frota: 128, capacidade: 5.60, litros: 716.80 },
  { marca: 'VOLVO', anoModelo: '2025 XC90 T8 PLUS', lubrificante: '0W20 C5', frota: 46, capacidade: 5.60, litros: 257.60 },
];
</script>

<template>
  <div class="container-fluid p-4">
    
    <div class="d-flex justify-content-between align-items-center mb-4">
      <div>
        <h4 class="fw-bold text-dark m-0">DASHBOARD MARCAS</h4>
        <small class="text-primary fw-bold text-uppercase">Capacidade por Marca</small>
      </div>
    </div>

    <div class="row g-4">
      
      <!-- === COLUNA ESQUERDA (FILTROS MARCA) === -->
      <div class="col-lg-3">
        <div class="d-flex flex-column gap-3 h-100">
            
            <!-- Botão Limpar Filtros (No topo igual ao print) -->
            <button class="btn btn-outline-secondary w-100 d-flex align-items-center justify-content-center gap-2 py-2">
                <Eraser :size="18" /> Limpar todos os filtros
            </button>

            <div class="card border-0 shadow-sm bg-white flex-grow-1">
                <div class="card-body d-flex flex-column">
                    
                    <!-- Carro/Moto -->
                    <div class="d-flex gap-2 mb-4">
                        <button class="btn flex-grow-1 py-2 border" :class="veiculoTipo === 'carro' ? 'btn-primary text-white' : 'btn-light text-muted'" @click="veiculoTipo = 'carro'">
                            <Car :size="28" />
                        </button>
                        <button class="btn flex-grow-1 py-2 border" :class="veiculoTipo === 'moto' ? 'btn-primary text-white' : 'btn-light text-muted'" @click="veiculoTipo = 'moto'">
                            <Bike :size="28" />
                        </button>
                    </div>

                    <!-- Slider Ano -->
                    <div class="mb-4 bg-light p-3 rounded">
                        <label class="fw-bold small text-muted mb-2">ANO</label>
                        <div class="d-flex gap-2 align-items-center">
                            <input type="number" class="form-control text-center fw-bold" v-model="anoInicio">
                            <span class="text-muted">-</span>
                            <input type="number" class="form-control text-center fw-bold" v-model="anoFim">
                        </div>
                        <input type="range" class="form-range mt-3" min="1990" max="2025" step="1">
                    </div>

                    <!-- CHECKLIST MARCA -->
                    <div class="flex-grow-1 d-flex flex-column min-h-0">
                        <div class="d-flex justify-content-between align-items-center mb-2">
                            <label class="fw-bold small text-muted">MARCA</label>
                            <small class="text-muted" style="font-size: 0.7rem;">{{ filteredMarcas.length }} encontradas</small>
                        </div>
                        
                        <!-- Busca rápida de marca -->
                        <div class="input-group input-group-sm mb-2">
                            <span class="input-group-text bg-light border-end-0"><Search :size="14" /></span>
                            <input type="text" class="form-control bg-light border-start-0" placeholder="Buscar..." v-model="marcaSearch">
                        </div>

                        <!-- Lista com Scroll -->
                        <div class="overflow-auto custom-scroll border rounded p-2 bg-light h-100" style="max-height: 400px;">
                            <div v-for="(marca, index) in filteredMarcas" :key="index" class="form-check mb-2">
                                <input class="form-check-input" type="checkbox" :id="'marca-'+index" v-model="marca.checked">
                                <label class="form-check-label small fw-bold text-secondary" :for="'marca-'+index">
                                    {{ marca.name }}
                                </label>
                            </div>
                        </div>
                    </div>

                </div>
            </div>
        </div>
      </div>

      <!-- === COLUNA DIREITA (CONTEÚDO IGUAL AO REGIAO) === -->
      <div class="col-lg-9">

        
        <!-- LINHA 2 (Filtros Horizontais) -->
        <div class="card border-0 shadow-sm mb-3 bg-dark text-white">
          <div class="card-body p-3">
            <div class="row g-2">
              <div class="col"><label class="small text-white-50 fw-bold">MARCA</label><select class="form-select form-select-sm bg-dark text-white border-secondary"><option>Todos</option></select></div>
              <div class="col"><label class="small text-white-50 fw-bold">ANO</label><select class="form-select form-select-sm bg-dark text-white border-secondary"><option>Todos</option></select></div>
              <div class="col"><label class="small text-white-50 fw-bold">MODELO</label><select class="form-select form-select-sm bg-dark text-white border-secondary"><option>Todos</option></select></div>
              <div class="col"><label class="small text-white-50 fw-bold">MOTOR</label><select class="form-select form-select-sm bg-dark text-white border-secondary"><option>Todos</option></select></div>
              <div class="col"><label class="small text-white-50 fw-bold">CÂMBIO</label><select class="form-select form-select-sm bg-dark text-white border-secondary"><option>Todos</option></select></div>
            </div>
          </div>
        </div>
        
        <!-- LINHA 1 -->
        <div class="row g-3 mb-3">
          <div class="col-lg-4">
            <div class="card border-0 shadow-sm h-100">
              <div class="card-body p-2 d-flex flex-column">
                <div class="mb-2">
                    <label class="small fw-bold text-muted">DISTRIBUIDOR / ESTADO</label>
                    <select class="form-select form-select-sm border-secondary"><option>Todos</option></select>
                </div>
                <div class="flex-grow-1 bg-light rounded position-relative overflow-hidden">
                   <div ref="mapContainer" class="w-100 h-100" style="min-height: 200px; border-radius: 0.5rem;"></div>
                </div>
              </div>
            </div>
          </div>

          <div class="col-lg-4">
            <div class="card border-0 shadow-sm h-100">
              <div class="card-body p-3">
                <h6 class="fw-bold small text-uppercase mb-3">Capacidade por Cidade</h6>
                <div style="height: 200px;"><Bar :data="cityChartData" :options="chartOptions" /></div>
              </div>
            </div>
          </div>

          <div class="col-lg-4">
            <div class="card border-0 shadow-sm h-100">
              <div class="card-body p-3">
                <h6 class="fw-bold small text-uppercase mb-3">Capacidade por Viscosidade</h6>
                <div style="height: 200px;"><Bar :data="viscosityChartData" :options="chartOptions" /></div>
              </div>
            </div>
          </div>
        </div>


        <!-- LINHA 3 (Tabela) -->
        <div class="card border-0 shadow-sm">
          <div class="card-header bg-white border-0 py-2 d-flex justify-content-between align-items-center">
            <span class="fw-bold small text-uppercase">Capacidade por Marca e Modelo</span>
            <span class="badge bg-secondary">37.720 REGISTROS</span>
          </div>
          <div class="table-responsive">
            <table class="table table-hover table-sm align-middle mb-0" style="font-size: 0.85rem;">
              <thead class="bg-dark text-white">
                <tr>
                  <th class="py-2 ps-3">MARCA</th>
                  <th class="py-2">ANO-MODELO</th>
                  <th class="py-2">LUBRIFICANTE</th>
                  <th class="py-2 text-end">FROTA</th>
                  <th class="py-2 text-end">CAPACIDADE</th>
                  <th class="py-2 text-end pe-3">LITROS</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(row, index) in tableData" :key="index">
                  <td class="fw-bold ps-3">{{ row.marca }}</td>
                  <td>{{ row.anoModelo }}</td>
                  <td>{{ row.lubrificante }}</td>
                  <td class="text-end fw-bold">{{ row.frota }}</td>
                  <td class="text-end">{{ row.capacidade }}</td>
                  <td class="text-end pe-3">{{ row.litros }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

      </div>
    </div>
  </div>
</template>

<style scoped>
/* Scrollbar customizada para a lista de marcas */
.custom-scroll::-webkit-scrollbar { width: 6px; }
.custom-scroll::-webkit-scrollbar-thumb { background-color: #cbd5e1; border-radius: 3px; }
.text-white-50 { color: rgba(255, 255, 255, 0.6) !important; }
</style>
