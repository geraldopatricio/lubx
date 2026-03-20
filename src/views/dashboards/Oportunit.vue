<script setup>
import { ref, computed, onMounted, watch } from 'vue';
import { Bar, Doughnut } from 'vue-chartjs';
import { 
  Chart as ChartJS, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale, ArcElement, PointElement, LineElement 
} from 'chart.js';

// Plugin necessário para os valores aparecerem nos gráficos (como os Mi e %)
import ChartDataLabels from 'chartjs-plugin-datalabels';

import { 
  Car, Bike, Truck, Calculator, Info, RotateCcw, Filter, MapPin, 
  ChevronDown, ChevronUp, MousePointer2, Eraser, List, ExternalLink, PieChart, LayoutDashboard, ArrowLeft 
} from 'lucide-vue-next'; 

import mapboxgl from 'mapbox-gl';
import 'mapbox-gl/dist/mapbox-gl.css';

// --- REGISTRO DO CHART.JS ---
ChartJS.register(
  CategoryScale, 
  LinearScale, 
  BarElement, 
  ArcElement, 
  PointElement, 
  LineElement, 
  Title, 
  Tooltip, 
  Legend, 
  ChartDataLabels // Plugin para labels externos
);

// --- ESTADOS ---
const isLoading = ref(true);
const selectedUF = ref('Todos');
const selectedStateName = ref('');

// Configuração do Simulador de Share (Sincronizado)
const simuladorShare = ref({
  venda: 50000,
  meses: 3,
  shareDesejado: 15.00
});

const simuladorTrocas = ref({ leve: 1.0, pesada: 3.8, moto: 3.0 });

// Funções de Reset (Ícones de Borracha)
const resetVenda = () => simuladorShare.value.venda = 0;
const resetMeses = () => simuladorShare.value.meses = 0;

// --- CONFIG GRÁFICOS ---

// Opção global para desativar labels em gráficos que não precisam
const chartOptionsBase = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
    datalabels: { display: false }
  }
};

// Gráfico de Rosca (Share x Oportunidade) com labels externos
const doughnutData = {
  labels: ['Sua Venda', 'Oportunidade'],
  datasets: [{
    data: [0.01, 99.99],
    backgroundColor: ['#e97332', '#cbd5e1'],
    borderWidth: 0,
  }]
};

const doughnutOptions = {
  responsive: true,
  maintainAspectRatio: false,
  cutout: '60%',
  layout: { padding: 30 },
  plugins: {
    legend: { display: false },
    datalabels: {
      display: true,
      anchor: 'end',
      align: 'end',
      offset: 8,
      color: '#666',
      font: { size: 10, weight: 'bold' },
      formatter: (val) => val.toString().replace('.', ',') + '%'
    }
  }
};

// Gráfico de Barras (Oportunidade por Viscosidade)
const barData = {
  labels: ['0W20', '0W30', '0W40', '10W30', '10W40'],
  datasets: [
    { label: 'Oportunidade', backgroundColor: '#cbd5e1', data: [30, 20, 10, 60, 45] },
    { label: 'Sua Venda', backgroundColor: '#e97332', data: [5, 2, 1, 15, 8] }
  ]
};

// --- MAPA ---
const mapContainer = ref(null);
const map = ref(null);

// --- DENTRO DO <script setup> ---


// --- LÓGICA PARA A TABELA DE DETALHAMENTO ---
const tableData = computed(() => {
  // Lista baseada nas labels do gráfico de barras do seu código
  const labels = ['0W20', '0W20 C5', '0W30 A3/B4', '0W30 A5/B5', '0W30 C2', '0W30 C3', '0W40', '0W40 A3/B4', '0W40 C3', '10W30', '10W30 O', '10W30 A3/B4', '10W30 E7', '10W30 E7/E9', '10W30 E9', '10W40', '10W40 O', '10W40 A3/B4', '10W40 A5/B5', '10W40 E6', '10W40 E7', '10W40 E7/E9', '10W40 E9', '10W50'];
  const potencialMi = [15, 2, 1, 1, 5, 2, 1, 2, 1, 55, 3, 1, 1, 1, 1, 35, 2, 5, 2, 4, 28, 2, 12, 2];
  
  // Cálculo simples de share proporcional para o simulador
  const totalPotencial = potencialMi.reduce((a, b) => a + b, 0) * 1000000;
  const currentShare = simuladorShare.value.venda > 0 ? (simuladorShare.value.venda / totalPotencial) : 0;

  return labels.map((label, index) => {
    const pot = potencialMi[index] * 1000000;
    const estVenda = pot * currentShare;
    const gap = pot - estVenda;
    return {
      especificacao: label,
      potencial: pot,
      vendaEst: estVenda,
      gap: gap > 0 ? gap : 0,
      share: (currentShare * 100).toFixed(1) + '%'
    };
  });
});

const marketShareCalculado = computed(() => {
  const potencialTotalAnual = 604442286; // Valor vindo do seu Overview
  if (!simuladorShare.value.venda || !simuladorShare.value.meses || simuladorShare.value.meses <= 0) return '0.00';
  
  // Ajusta o potencial anual para o período de meses digitado
  const potencialNoPeriodo = (potencialTotalAnual / 12) * simuladorShare.value.meses;
  const share = (simuladorShare.value.venda / potencialNoPeriodo) * 100;
  
  return share.toFixed(2);
});

// 1. Atualize a lógica do onMounted para o Mapa com Hover
onMounted(() => {
  // Esta é a linha que você deve usar:
  mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_TOKEN;
  
  map.value = new mapboxgl.Map({
    container: mapContainer.value,
    style: 'mapbox://styles/mapbox/light-v11',
    center: [-52, -15],
    zoom: 3,
    attributionControl: false
  });

  map.value.on('load', () => {
    // Adiciona a fonte dos estados
    map.value.addSource('states', {
      type: 'geojson',
      data: 'https://raw.githubusercontent.com/codeforamerica/click_that_hood/master/public/data/brazil-states.geojson',
      generateId: true // Necessário para o feature-state funcionar
    });

    // Camada de preenchimento (Fill)
    map.value.addLayer({
      id: 'states-fill',
      type: 'fill',
      source: 'states',
      paint: {
        'fill-color': '#e97332',
        // Se o mouse estiver sobre (hover), opacidade 0.6, senão 0.1
        'fill-opacity': [
          'case',
          ['boolean', ['feature-state', 'hover'], false],
          0.6,
          0.1
        ]
      }
    });

    // Camada de borda (Borders)
    map.value.addLayer({
      id: 'states-borders',
      type: 'line',
      source: 'states',
      paint: {
        'line-color': '#e97332',
        'line-width': 1
      }
    });

    let hoveredStateId = null;

    // Lógica de Mouse Move para o destaque
    map.value.on('mousemove', 'states-fill', (e) => {
      if (e.features.length > 0) {
        if (hoveredStateId !== null) {
          map.value.setFeatureState(
            { source: 'states', id: hoveredStateId },
            { hover: false }
          );
        }
        hoveredStateId = e.features[0].id;
        map.value.setFeatureState(
          { source: 'states', id: hoveredStateId },
          { hover: true }
        );
        mapContainer.value.style.cursor = 'pointer';
      }
    });

    // Lógica de Mouse Leave para resetar o destaque
    map.value.on('mouseleave', 'states-fill', () => {
      if (hoveredStateId !== null) {
        map.value.setFeatureState(
          { source: 'states', id: hoveredStateId },
          { hover: false }
        );
      }
      hoveredStateId = null;
      mapContainer.value.style.cursor = '';
    });
  });

  setTimeout(() => { isLoading.value = false; }, 800);
});

// Formatação de números para exibição
const fmtNum = (v) => new Intl.NumberFormat('pt-BR').format(Math.floor(v));
</script>

<template>
  <div class="dashboard-wrapper">
    <div class="top-filter-bar shadow-sm px-4 py-2 bg-white d-flex align-items-center gap-4">
      
      <div v-for="f in ['VISCOSIDADE', 'API', 'ACEA', 'JASO', 'BÁSICO']" :key="f" class="filter-item">
        <label>{{ f }}:</label>
        <select class="form-select form-select-sm" style="width: auto; min-width: 100px;">
          <option>Todos</option>
        </select>
      </div>
      <button class="btn btn-red-total btn-sm ms-auto fw-bold px-4">Limpar todos os filtros</button>
    </div>

    <div class="main-content d-flex p-3 gap-3">
      
      <!-- 2. SIDEBAR ESQUERDA -->
      <div class="sidebar d-flex flex-column gap-3">
      
        <!-- Card Filtros UF -->
        <div class="card border-0 shadow-sm p-3 rounded-4 bg-white">
            <div class="d-flex align-items-center gap-2 mb-3">
                <Filter :size="18" class="text-dark" />
                <h6 class="fw-bold m-0 uppercase">Filtros</h6>
            </div>
            
            <div class="mb-3">
                <label class="small fw-bold text-muted d-block mb-1">ESTADO</label>
                <select class="form-select form-select-sm rounded-3">
                <option>Todos</option>
                </select>
            </div>

            <div>
              <label class="small fw-bold text-muted d-block mb-1">TIPO DE VEÍCULO</label>
              <div class="btn-group w-100 shadow-sm" role="group">
                  <button class="btn btn-vehicle-type active">
                      <Car :size="18" class="mb-1" />
                      <span class="btn-label">LEVE</span>
                  </button>
                  <button class="btn btn-vehicle-type">
                      <Bike :size="18" class="mb-1" />
                      <span class="btn-label">MOTO</span>
                  </button>
                  <button class="btn btn-vehicle-type">
                      <Truck :size="18" class="mb-1" />
                      <span class="btn-label">PESADA</span>
                  </button>
              </div>
        </div>
      </div>

        <div class="card border-0 shadow-sm rounded-4 bg-white overflow-hidden" style="height: 250px;">
    <div ref="mapContainer" class="w-100 h-100"></div>
  </div>



            <div class="card border-0 shadow-sm p-3 rounded-4 simulator-card-total"> 
    
            <!-- Header do Card -->
            <div class="d-flex justify-content-between align-items-center mb-2">
                <div class="d-flex align-items-center gap-2">
                    <Calculator :size="18" class="text-dark" />
                    <h6 class="fw-bold m-0 uppercase" style="color: #1a2332; font-size: 11px;">Simulador de Share</h6>
                </div>
                <div class="d-flex gap-1">
                    <div class="bg-white border rounded p-1"><Info :size="12" class="text-primary"/></div>
                    <div class="bg-white border rounded p-1"><List :size="12" class="text-muted"/></div>
                    <div class="bg-white border rounded p-1"><ExternalLink :size="12" class="text-muted"/></div>
                </div>
            </div>

            <!-- Texto de Instrução 1 - Fonte Padronizada -->
            <p class="text-muted mb-3" style="line-height: 1.2; font-size: 11px;">
                Insira a <span style="color: #e67e22; font-weight: 800;">VENDA ATUAL</span> em litros e o 
                <span style="color: #e67e22; font-weight: 800;">NÚMERO DE MESES CORRIDO</span> para projetar seu Market Share.
            </p>

            <!-- Linha com Venda e Meses Lado a Lado -->
            <div class="row g-2 mb-3">
                <div class="col-8">
                    <label class="fw-bold text-dark mb-1" style="font-size: 11px;">SUA VENDA</label>
                    <div class="pbi-input-custom">
                        <span class="pbi-input-label">Lts</span>
                        <input type="number" v-model="simuladorShare.venda" class="pbi-input-field">
                    </div>
                </div>
                <div class="col-4">
                    <label class="fw-bold text-dark mb-1" style="font-size: 11px;">MESES</label>
                    <input type="number" v-model="simuladorShare.meses" class="form-control form-control-sm fw-bold" style="height: 32px; font-size: 12px;">
                </div>
            </div>

            <!-- Campo SHARE DESEJADO -->
            <div class="mb-1">
            <!-- NOVA LINHA: Market Share calculado em tempo real -->
            <div class="d-flex align-items-center justify-content-between mb-3 mt-2">
                <span class="fw-bold text-dark" style="font-size: 11px; letter-spacing: 0.5px;">MARKET SHARE:</span>
                <div class="bg-white border rounded-pill px-3 py-1 shadow-sm d-flex align-items-center">
                    <span class="fw-black text-dark" style="font-size: 14px;">{{ marketShareCalculado }}%</span>
                </div>
            </div>

            <!-- Texto de Instrução 2 -->
            <p class="text-muted mb-2" style="line-height: 1.2; font-size: 11px;">
                Insira <span style="color: #e67e22; font-weight: 800;">SHARE DESEJADO</span> para projetar seu Market Share desejado.
            </p>
            
            <!-- Box de Valor Centralizado -->
            <div class="bg-white border rounded p-1 mb-2 d-flex align-items-center justify-content-center" style="height: 45px;">
                <input type="number" 
                        v-model.number="simuladorShare.shareDesejado" 
                        class="form-control form-control-sm fw-bold border-0 text-center" style="font-size: 16px;">
                <span class="fw-bold fs-5 pe-2">%</span>
            </div>
        </div>
        </div>


        

      </div>

      <!-- 3. PAINEL CENTRAL / DIREITO -->
      <div class="content-grid flex-grow-1 d-flex flex-column gap-3">
        
        <!-- Linha Superior -->
        <div class="row g-3">
          <!-- Simulador de Trocas -->
          <div class="col-lg-8">
            <!-- SIMULADOR DE TROCAS ANUAL POR TIPO DE VEÍCULO -->
            <div class="card border-0 shadow-sm p-4 rounded-4 bg-white ">
            <div class="d-flex align-items-center gap-2 mb-2">
                <Calculator :size="18" class="text-dark" />
                <h6 class="fw-bold m-0 uppercase" style="color: #1a2332;">Simulador de Trocas Anual por Tipo de Veículo</h6>
            </div>
            
            <p class="text-muted mb-4" style="font-size: 11px; line-height: 1.4;">
                Insira ou arraste o slice com a quantidade de trocas de óleo por ano de acordo com o tipo de veículos para estimar o potencial de consumo anual de acordo com a frota eletiva.
            </p>

            <div class="row g-0">
                <!-- Linha Leve -->
                <div class="col-4 border-end px-3">
                <div class="d-flex align-items-center gap-3 mb-3">
                    <Car class="text-orange" :size="48" />
                    <div>
                    <div class="fw-bold fs-5 text-dark">58.687.607</div>
                    <div class="text-muted smaller fw-medium">Veículos</div>
                    </div>
                </div>
                <div class="mb-4">
                    <div class="fw-bold fs-5 text-dark">229.911.597,38</div>
                    <div class="text-muted smaller fw-medium">Litros / ano</div>
                </div>
                <div class="border-top pt-3">
                    <label class="smaller text-muted d-block mb-1 fw-bold">Trocas por Ano</label>
                    <input type="text" class="form-control form-control-sm border-light-subtle bg-light-subtle" style="width: 85px" value="1,0">
                </div>
                </div>

                <!-- Linha Pesada -->
                <div class="col-4 border-end px-3">
                <div class="d-flex align-items-center gap-3 mb-3">
                    <Truck class="text-orange" :size="48" />
                    <div>
                    <div class="fw-bold fs-5 text-dark">3.797.561</div>
                    <div class="text-muted smaller fw-medium">Veículos</div>
                    </div>
                </div>
                <div class="mb-4">
                    <div class="fw-bold fs-5 text-dark">269.004.060,74</div>
                    <div class="text-muted smaller fw-medium">Litros / ano</div>
                </div>
                <div class="border-top pt-3">
                    <label class="smaller text-muted d-block mb-1 fw-bold">Trocas por Ano</label>
                    <input type="text" class="form-control form-control-sm border-light-subtle bg-light-subtle" style="width: 85px" value="3,8">
                </div>
                </div>

                <!-- Linha Moto -->
                <div class="col-4 px-3">
                <div class="d-flex align-items-center gap-3 mb-3">
                    <Bike class="text-orange" :size="48" />
                    <div>
                    <div class="fw-bold fs-5 text-dark">34.307.995</div>
                    <div class="text-muted smaller fw-medium">Veículos</div>
                    </div>
                </div>
                <div class="mb-4">
                    <div class="fw-bold fs-5 text-dark">105.526.627,98</div>
                    <div class="text-muted smaller fw-medium">Litros / ano</div>
                </div>
                <div class="border-top pt-3">
                    <label class="smaller text-muted d-block mb-1 fw-bold">Trocas por Ano</label>
                    <input type="text" class="form-control form-control-sm border-light-subtle bg-light-subtle" style="width: 85px" value="3,0">
                </div>
                </div>
            </div>

            <div class="mt-4 d-flex align-items-start gap-2">
                <Info :size="14" class="text-muted" />
                <p class="smaller text-muted m-0" style="line-height: 1.4;">
                A litragem total apresentada está alinhada com a estimativa média nacional de trocas de óleo por segmento.
                </p>
            </div>
            </div>
          </div>

          
        <!-- OVERVIEW (CARD BIPARTIDO: AZUL E LARANJA) -->
          <div class="col-lg-4">
            <div class="card border-0 shadow-sm rounded-4 overview-card text-white d-flex flex-column overflow-hidden" style="background-color: #1a2332;">
              
              <!-- PARTE SUPERIOR (AZUL ESCURO) -->
              <div class="p-3 flex-grow-1">
                <div class="d-flex align-items-center gap-2">
                  <Calculator :size="20" class="text-white" />
                  <h6 class="fw-bold m-0 uppercase" style="letter-spacing: 1px; font-size: 14px;">OVERVIEW</h6>
                </div>

                <div class="mb-4">
                  <p class="mb-1 fw-bold opacity-75" style="color: #cbd5e1; font-size: 14px;">Potencial de Consumo (L / Ano)</p>
                  <h1 class="fw-bold m-0" style="font-size: 34px; letter-spacing: -1px;">604.442.286</h1>
                </div>

                <div class="mb-0">
                  <h3 class="fw-bold m-0" style="font-size: 26px;">0</h3>
                  <p class="mb-0 fw-bold opacity-75" style="color: #cbd5e1; font-size: 12px; margin-top: 4px;">Sua Venda Projetada</p>
                </div>
              </div>

              <!-- PARTE INFERIOR (LARANJA #e97332) -->
              <div class="p-3" style="background-color: #e97332;">
                <div class="mb-4">
                  <h3 class="fw-bold m-0" style="font-size: 26px;">0,00%</h3>
                  <p class="mb-0 fw-bold text-white opacity-100" style="font-size: 12px; margin-top: 4px;">Share Projetado</p>
                </div>

                <div class="mt-auto">
                  <h2 class="fw-bold m-0" style="font-size: 30px;">96.793.163</h2>
                  <p class="mb-0 fw-bold text-white opacity-100" style="font-size: 13px; margin-top: 5px;">Frota Eletiva</p>
                </div>
              </div>

            </div>
          </div>


        </div>

        <!-- Linha Inferior -->
        <div class="row g-3 flex-grow-1">
          

        <!-- TABELA DE DETALHAMENTO (DATAGRID) -->
        <div class="row g-2 mt-1">

          <div class="col-12">
            <div class="card border-0 shadow-sm rounded-4 bg-white overflow-hidden">
              <div class="card-header bg-white border-0 py-3 px-4 d-flex justify-content-between align-items-center">
                <h6 class="fw-bold m-0 uppercase" style="color: #1a2332; font-size: 14px;">DETALHAMENTO POR ESPECIFICAÇÃO</h6>
                <span class="badge rounded-pill bg-light text-dark border px-3">{{ tableData.length }} Registros</span>
              </div>
              
              <div class="table-responsive" style="max-height: 400px;">
                <table class="table table-hover align-middle mb-0">
                  <thead class="sticky-top bg-white" style="z-index: 10;">
                    <tr class="border-bottom">
                      <th class="ps-4 py-3 text-muted fw-bold small">ESPECIFICAÇÃO</th>
                      <th class="text-end py-3 text-muted fw-bold small">POTENCIAL (L)</th>
                      <th class="text-end py-3 text-orange fw-bold small" style="width: 180px;">SUA VENDA (EST.)</th>
                      <th class="text-end py-3 text-muted fw-bold small">GAP (LITROS)</th>
                      <th class="text-end pe-4 py-3 text-muted fw-bold small">SHARE</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="(row, idx) in tableData" :key="idx" class="border-bottom">
                      <td class="ps-4 fw-bold text-dark">{{ row.especificacao }}</td>
                      <td class="text-end fw-bold">{{ fmtNum(row.potencial) }}</td>
                      <td class="text-end">
                        <div class="d-flex justify-content-end">
                          <input type="text" 
                                 class="form-control form-control-sm text-end fw-bold border-orange-subtle bg-white" 
                                 style="max-width: 100px; color: #e97332;"
                                 :value="Math.floor(row.vendaEst)">
                        </div>
                      </td>
                      <td class="text-end text-muted">{{ fmtNum(row.gap) }}</td>
                      <td class="text-end pe-4">
                        <span class="badge bg-light text-dark border fw-bold" style="font-size: 11px;">{{ row.share }}</span>
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        </div>

        </div>

      </div>
    </div>
  </div>
</template>

<style scoped>
.dashboard-wrapper {
  background-color: #f1f5f9;
  height: 120vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  font-family: 'Inter', sans-serif;
}

.uppercase { text-transform: uppercase; letter-spacing: 0.5px; font-size: 13px; }
.smaller { font-size: 10px; }
.text-orange { color: #e97332 !important; }
.bg-white { background-color: #ffffff; }

/* BARRA SUPERIOR */
.top-filter-bar { height: 60px; z-index: 100; }


.filter-item { 
  display: flex; 
  align-items: center; /* Alinha verticalmente no centro */
  gap: 8px;            /* Espaço entre o título e o select */
}
.filter-item label { 
  font-size: 9px; 
  font-weight: 800; 
  color: #334155; 
  white-space: nowrap; /* Impede que o texto quebre linha */
  margin-bottom: 0;    /* Remove margem inferior que empurra o select */
}


.form-select-sm { padding: 0.1rem 1.5rem 0.1rem 0.5rem; font-size: 11px; }
.btn-red-total { background-color: #ff0000; color: white; border: none; }

/* SIDEBAR */
.sidebar { width: 300px; flex-shrink: 0; }
.simulator-card { 
  background: linear-gradient(135deg, #fff3e6 0%, #ffe9cc 100%); 
  border-left: 5px solid #e97332 !important;
}

/* CONTENT GRID */
.content-grid { 
    overflow-y: auto; 
    overflow-x: hidden; /* Isso remove a barra horizontal indesejada */
    padding-right: 8px; /* Espaço para a barra vertical não colar no card */
    width: 100%;
    padding-bottom: 100px !important;
}

.table-responsive {
    max-height: 500px; /* Você pode aumentar um pouco se desejar */
    overflow-y: auto;
}

.troca-input { width: 80px; }
.chart-center-text {
  position: absolute;
  top: 55%; left: 50%;
  transform: translate(-50%, -50%);
  font-weight: 800; font-size: 16px;
}
.dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
.dot.orange { background: #e97332; }
.dot.grey { background: #cbd5e1; }

.overview-card { background-color: #1a2332; }
.fw-black { font-weight: 900; }

/* CUSTOM RANGE */
.custom-range::-webkit-slider-thumb { background: #e97332; }

@media (max-width: 1400px) {
  .dashboard-wrapper { overflow-y: auto; height: auto; }
  .main-content { flex-direction: column; }
  .sidebar { width: 100%; }
}

/* Estilo do Card Simulador */
.simulator-card-total {
  background-color: #fff5e6; /* Amarelo bem claro do print */
  border: 1px solid #f1c40f !important; /* Borda amarela fina */
}

/* Estilo dos Inputs Estilo PBI */
.pbi-input-container {
  border: 1px solid #ccc;
  background: white;
  border-radius: 4px;
}

.pbi-input {
  height: 45px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-weight: 600;
}

.pbi-input-small {
  width: 100px;
  height: 35px;
  border: 1px solid #ccc;
}

/* Ícones de ação dentro do input */
.input-actions-pbi {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  display: flex;
  align-items: center;
  gap: 8px;
  background: white;
  padding-left: 5px;
}

.pbi-icon-action {
  color: #666;
  cursor: pointer;
}
.pbi-icon-action:hover { color: #e67e22; }

.pbi-icon-step {
  cursor: pointer;
  color: #333;
}

/* Estilo do Slider (Bolinha Branca com Borda) */
.pbi-slider {
  -webkit-appearance: none;
  height: 4px;
  background: #ddd;
  border-radius: 5px;
}

.pbi-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 24px;
  height: 24px;
  background: white;
  border: 2px solid #555;
  border-radius: 50%;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

.cursor-pointer { cursor: pointer; }
.uppercase { text-transform: uppercase; }

/* Botões de visualização no topo do card */
.btn-chart-view {
  width: 34px;
  height: 34px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
  color: #64748b;
  cursor: pointer;
  background: white;
}

.btn-chart-view.active {
  background: #f8fafc;
  color: #1a2332;
  border-color: #cbd5e1;
  box-shadow: inset 0 2px 4px rgba(0,0,0,0.05);
}



.btn-vehicle-type {
    border: 1px solid #dee2e6;
    background-color: #fff;
    color: #6c757d;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 8px 0;
    width: 100%;
    transition: all 0.2s;
}

/* Estado Ativo com a cor solicitada */
.btn-vehicle-type.active {
    background-color: #e97332 !important;
    border-color: #e97332 !important;
    color: white !important;
}

.btn-vehicle-type:hover:not(.active) {
    background-color: #fff1eb;
    color: #e97332;
}

.btn-label {
    font-size: 9px;
    font-weight: 700;
}

.table-responsive table th { 
    font-size: 10px !important; 
    padding-top: 8px !important; 
    padding-bottom: 8px !important; 
}

.table-responsive table td { 
    font-size: 11px !important; 
    padding-top: 4px !important; 
    padding-bottom: 4px !important; 
}

/* Ajuste do input dentro da tabela para acompanhar a fonte */
.table-responsive .form-control-sm { 
    font-size: 11px !important; 
    height: 25px !important; 
}


/* Estilo Customizado para o Input com prefixo Lts */
.pbi-input-custom {
  display: flex;
  align-items: center;
  background: white;
  border: 1px solid #dee2e6;
  border-radius: 50px; /* Formato de pílula/arredondado */
  overflow: hidden;
  height: 32px;
  transition: all 0.2s ease-in-out;
}

/* Efeito de foco (borda laranja igual ao seu print) */
.pbi-input-custom:focus-within {
  border-color: #e97332;
  box-shadow: 0 0 0 3px rgba(233, 115, 50, 0.15);
}

.pbi-input-label {
  background-color: #f8fafc;
  color: #1a2332;
  font-weight: 800;
  font-size: 11px;
  padding: 0 12px;
  height: 100%;
  display: flex;
  align-items: center;
  border-right: 1px solid #dee2e6;
}

.pbi-input-field {
  border: none;
  flex: 1;
  padding: 0 10px;
  font-size: 12px;
  font-weight: 700;
  color: #334155;
  outline: none;
  background: transparent;
}

/* Remove as setinhas padrão do campo de número para ficar mais limpo */
.pbi-input-field::-webkit-inner-spin-button,
.pbi-input-field::-webkit-outer-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

</style>
