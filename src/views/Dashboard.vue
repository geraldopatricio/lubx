<script setup>
import { ref, computed, onMounted, watch } from 'vue';
import { Bar, Doughnut, Pie } from 'vue-chartjs';
import { 
  Chart as ChartJS, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale, RadialLinearScale, ArcElement, PointElement, LineElement, BarController, LineController
} from 'chart.js';
import ChartDataLabels from 'chartjs-plugin-datalabels';
import { 
  Car, Bike, Truck, Info, ChevronLeft, ChevronRight, Globe, X, Filter, Clock, TrendingUp, MapPin, DollarSign, Zap, Map as MapIcon, ArrowLeft
} from 'lucide-vue-next'; 
import mapboxgl from 'mapbox-gl';
import 'mapbox-gl/dist/mapbox-gl.css';

// --- REGISTRO DO CHART.JS ---
ChartJS.register(
    CategoryScale, 
    LinearScale, 
    RadialLinearScale, 
    BarElement, 
    ArcElement, 
    PointElement, 
    LineElement, 
    BarController,
    LineController,
    Title, 
    Tooltip, 
    Legend, 
    ChartDataLabels // Plugin para valores nos gráficos
);

// --- ESTADOS DE CONTROLE ---
const viewMode = ref('executive'); 
const selectedUF = ref('Todos');
const selectedStateName = ref('');
const showModal = ref(false);
const modalTitle = ref('');
const hintInfo = ref({ visible: false, x: 0, y: 0, uf: '', frota: 0 });

const ufs = ['AC', 'AL', 'AP', 'AM', 'BA', 'CE', 'DF', 'ES', 'GO', 'MA', 'MT', 'MS', 'MG', 'PA', 'PB', 'PR', 'PE', 'PI', 'RJ', 'RN', 'RS', 'RO', 'RR', 'SC', 'SP', 'SE', 'TO'];
const stateNames = {
  'AC': 'Acre', 'AL': 'Alagoas', 'AP': 'Amapá', 'AM': 'Amazonas', 'BA': 'Bahia', 'CE': 'Ceará', 'DF': 'Distrito Federal', 'ES': 'Espírito Santo', 'GO': 'Goiás', 'MA': 'Maranhão', 'MT': 'Mato Grosso', 'MS': 'Mato Grosso do Sul', 'MG': 'Minas Gerais', 'PA': 'Pará', 'PB': 'Paraíba', 'PR': 'Paraná', 'PE': 'Pernambuco', 'PI': 'Piauí', 'RJ': 'Rio de Janeiro', 'RN': 'Rio Grande do Norte', 'RS': 'Rio Grande do Sul', 'RO': 'Rondônia', 'RR': 'Roraima', 'SC': 'Santa Catarina', 'SP': 'São Paulo', 'SE': 'Sergipe', 'TO': 'Tocantins'
};

const UF_COORDS = {
  'AC': [-70.55, -9.02], 'AL': [-36.78, -9.57], 'AP': [-52.00, 0.90], 'AM': [-65.00, -3.41], 'BA': [-41.70, -12.57], 'CE': [-39.60, -5.20], 'DF': [-47.92, -15.78], 'ES': [-40.30, -19.18], 'GO': [-49.26, -15.93], 'MA': [-45.00, -5.00], 'MT': [-56.09, -12.68], 'MS': [-54.50, -20.77], 'MG': [-44.38, -18.51], 'PA': [-52.00, -3.50], 'PB': [-36.50, -7.23], 'PR': [-51.50, -24.60], 'PE': [-37.00, -8.00], 'PI': [-42.50, -7.71], 'RJ': [-42.50, -22.00], 'RN': [-36.50, -5.79], 'RS': [-53.00, -30.00], 'RO': [-62.50, -11.50], 'RR': [-61.30, 2.82], 'SC': [-50.00, -27.00], 'SP': [-48.00, -22.00], 'SE': [-37.50, -10.57], 'TO': [-48.33, -10.16]
};

// --- INSIGHTS (TELA 1) ---
const currentInsights = computed(() => {
  const isBrazil = selectedUF.value === 'Todos';
  return [
    { label: isBrazil ? 'Frota Elegível' : `Frota em ${selectedUF.value}`, value: isBrazil ? '14,7M' : '2,1M', icon: Car },
    { label: isBrazil ? 'Sudeste concentra' : 'Market Share', value: isBrazil ? '48%' : '12.4%', icon: Globe },
    { label: isBrazil ? 'Diesel na frota' : 'Ticket Médio', value: isBrazil ? '27%' : 'R$ 452', icon: Truck },
    { label: isBrazil ? 'Idade média' : 'Potencial Lub.', value: isBrazil ? '12,5 anos' : '850k', icon: Clock }
  ];
});

// --- OPÇÕES E DADOS GRÁFICOS ---

// Cores do Print
const orangePalette = ['#e97332', '#f4b393', '#fbe2d5'];
const normasPalette = ['#007bff', '#1a237e', '#4a148c', '#880e4f', '#b71c1c', '#e65100', '#f57c00', '#ffb300', '#fbc02d', '#4caf50', '#009688', '#00bcd4'];

// Opções para Roscas (Com labels externas)
const doughnutOptionsExtended = {
    responsive: true,
    maintainAspectRatio: false,
    layout: { padding: 35 },
    plugins: {
        legend: { display: false },
        datalabels: {
            display: true,
            anchor: 'end',
            align: 'end',
            offset: 8,
            color: '#666',
            font: { size: 10, weight: 'bold' },
            formatter: (value) => value.toFixed(2).replace('.', ',') + '%'
        }
    },
    cutout: '60%'
};

const normasOptions = {
    responsive: true,
    maintainAspectRatio: false,
    layout: { padding: 45 }, // Aumentei um pouco o padding para caber os números de litros
    plugins: {
        legend: { display: false },
        datalabels: {
            display: true,
            anchor: 'end',
            align: 'end',
            offset: 8,
            color: '#666',
            font: { size: 10, weight: 'bold' },
            // Formata para padrão brasileiro (ex: 37,01) sem o símbolo de %
            formatter: (value) => value.toLocaleString('pt-BR', { minimumFractionDigits: 2 })
        }
    },
    cutout: '60%'
};

const basicData = { 
    labels: ['MINERAL', 'SINTÉTICO', 'SEMISSINTÉTICO'], 
    datasets: [{ data: [42.43, 17.96, 39.59], backgroundColor: orangePalette, borderWidth: 0 }] 
};

const segmentoData = { 
    labels: ['LEVE', 'MOTO', 'PESADA'], 
    datasets: [{ data: [33.33, 33.33, 33.33], backgroundColor: orangePalette, borderWidth: 0 }] 
};

const normasData = { 
    labels: ['VW', 'MB', 'FIAT', 'GM', 'FORD', 'OUTROS'], 
    datasets: [{ 
        data: [37.01, 14.03, 7.39, 5.62, 3.42, 2.7, 2.17, 1.51, 0.97, 0.82, 0.55, 0.3], 
        backgroundColor: normasPalette, 
        borderWidth: 0 
    }] 
};

const marcasMockData = {
    labels: ['Volkswagen', 'Toyota', 'Fiat', 'GM/Chevrolet', 'Ford', 'Honda', 'Hyundai'],
    datasets: [{
        data: [25, 20, 18, 12, 10, 8, 7],
        backgroundColor: ['#e97332', '#f4b393', '#fbe2d5', '#3b82f6', '#1d4ed8', '#10b981', '#f59e0b'],
        borderWidth: 0
    }]
};

const modelosMockData = {
    labels: ['Gol', 'Corolla', 'Onix', 'Strada', 'HB20', 'Civic', 'Hilux'],
    datasets: [{
        data: [22, 19, 15, 14, 12, 10, 8],
        backgroundColor: ['#e97332', '#f4b393', '#fbe2d5', '#3b82f6', '#1d4ed8', '#10b981', '#f59e0b'],
        borderWidth: 0
    }]
};

const modalChartData = computed(() => {
    if (modalTitle.value === 'Marcas') return marcasMockData;
    if (modalTitle.value === 'Modelos') return modelosMockData;
    return basicData;
});

// Opções padrão para outros gráficos (Desativa labels onde não queremos)
const chartOptionsBase = { 
    responsive: true, 
    maintainAspectRatio: false, 
    plugins: { 
        legend: { display: false },
        datalabels: { display: false } 
    }, 
    scales: { 
        y: { beginAtZero: true, grid: { borderDash: [2, 2], color: '#e2e8f0' }, ticks: { font: { size: 9 } } }, 
        x: { grid: { display: false }, ticks: { font: { size: 9 } } } 
    } 
};

// --- FUNÇÕES ---
const fmtNum = (v) => new Intl.NumberFormat('pt-BR').format(Math.floor(v));

const resetToBrazil = () => {
  selectedUF.value = 'Todos';
  selectedStateName.value = '';
  hintInfo.value.visible = false;
  if(map.value) {
    map.value.flyTo({ center: [-52, -15], zoom: 3.5, duration: 1000 });
    updateMapHighlight();
  }
};

const updateMapHighlight = () => {
    if(!map.value) return;
    map.value.setPaintProperty('states-fill', 'fill-color', ['case', ['==', ['get', 'sigla'], selectedUF.value], '#e97332', '#f97316']);
    map.value.setPaintProperty('states-fill', 'fill-opacity', ['case', ['==', ['get', 'sigla'], selectedUF.value], 0.5, 0.1]);
    map.value.setPaintProperty('states-borders', 'line-width', ['case', ['==', ['get', 'sigla'], selectedUF.value], 3, 0.5]);
};

// --- MAPBOX ---
const mapContainer = ref(null);
const map = ref(null);

const MAPBOX_TOKEN = import.meta.env.VITE_MAPBOX_TOKEN;

const initMap = () => {
    mapboxgl.accessToken = MAPBOX_TOKEN;
    map.value = new mapboxgl.Map({ container: mapContainer.value, style: 'mapbox://styles/mapbox/light-v11', center: [-52, -15], zoom: 3.5, projection: 'mercator', attributionControl: false });
    map.value.on('load', () => {
        map.value.addSource('brazil-states', { type: 'geojson', data: 'https://raw.githubusercontent.com/codeforamerica/click_that_hood/master/public/data/brazil-states.geojson', generateId: true });
        map.value.addLayer({ 'id': 'states-fill', 'type': 'fill', 'source': 'brazil-states', 'paint': { 'fill-color': '#f97316', 'fill-opacity': 0.1 }});
        map.value.addLayer({ 'id': 'states-borders', 'type': 'line', 'source': 'brazil-states', 'paint': { 'line-color': '#e97332', 'line-width': 0.5 }});
        
        map.value.on('click', (e) => {
            const features = map.value.queryRenderedFeatures(e.point, { layers: ['states-fill'] });
            if (features.length > 0) {
                const props = features[0].properties;
                selectedUF.value = props.sigla;
                selectedStateName.value = props.name || stateNames[props.sigla];
                updateMapHighlight();
            } else { resetToBrazil(); }
        });
    });
};

onMounted(() => initMap());
watch(selectedUF, () => updateMapHighlight());
</script>

<template>
  <div class="dashboard-container">
    
    <!-- TELA 1: PANORAMA (PRESERVADO VERSÃO 3) -->
    <div v-show="viewMode === 'executive'" class="view-wrapper p-4">
      <div class="row g-4 align-items-stretch">
        <div class="col-lg-7">
          <div class="card map-card h-100 position-relative border-0 shadow-sm rounded-4 overflow-hidden bg-white">
            <div class="p-4 position-absolute top-0 start-0 z-2">
              <h3 class="fw-bold m-0" style="color: #e97332">Panorama <span class="text-dark">Estratégico por Região</span></h3>
              <p class="text-muted small">Selecione a região ou clique <a href="#" @click.prevent="resetToBrazil" class="text-primary">AQUI</a> para o Brasil todo</p>
            </div>
            <div ref="mapContainer" class="w-100 h-100" style="min-height: 700px;"></div>
          </div>
        </div>


        <div class="col-lg-5">
            <div class="card h-100 border-0 shadow-sm p-4 d-flex flex-column rounded-4 bg-white">
                <div class="d-flex align-items-center gap-3 mb-4">
                <div class="icon-header"><MapIcon class="text-white" :size="24"/></div>
                <h2 class="fw-bold m-0">{{ selectedStateName || 'Brasil' }}</h2>
                </div>
                <div class="row mb-4">
                    <div class="col-6"><h3 class="fw-bold m-0">604.442.286</h3><small class="text-muted">Litros / ano total</small></div>
                    <div class="col-6 border-start ps-4"><h3 class="fw-bold m-0">96.793.163</h3><small class="text-muted">frota Geral</small></div>
                </div>
                <div class="info-alert p-3 rounded-3 mb-4 d-flex gap-2">
                    <Info :size="18" class="text-muted mt-1"/>
                    <small class="text-muted">Litragem alinhada com estimativa média nacional de trocas por segmento.</small>
                </div>
                <hr class="mb-4">
                
                <div class="flex-grow-1">
                    <label class="fw-bold mb-4 d-block">Segmentação e Perfil:</label>
                    
                    <!-- Segmentos Existentes -->
                    <div v-for="s in [{l:'Leve', i:Car, p:'60,63%', v:'229.911.597', f:'58.687.607'}, {l:'Pesada', i:Truck, p:'15,92%', v:'269.004.060', f:'3.797.561'}, {l:'Moto', i:Bike, p:'35,44%', v:'105.526.627', f:'34.307.995'}]" :key="s.l" class="mb-4">
                        <div class="d-flex align-items-start gap-3">
                            <component :is="s.i" class="text-orange mt-1" :size="32"/>
                            <div class="flex-grow-1">
                                <div class="d-flex justify-content-between"><strong class="fs-6">{{ s.v }}</strong><strong>{{ s.f }}</strong></div>
                                <div class="d-flex align-items-center gap-2">
                                    <div class="progress flex-grow-1" style="height: 4px;"><div class="progress-bar bg-orange" :style="{width: s.p}"></div></div>
                                    <small class="fw-bold" style="font-size: 10px;">{{ s.p }}</small>
                                </div>
                                <small class="text-muted" style="font-size: 10px;">Linha {{ s.l }}</small>
                            </div>
                        </div>
                    </div>

                    <!-- Novo Bloco: Idade Média (Inserido abaixo de Moto) -->
                    <div class="mb-4 pt-3 border-top">
                        <div class="d-flex align-items-start gap-3">
                            <Clock class="text-orange mt-1" :size="32"/>
                            <div class="flex-grow-1">
                                <div class="d-flex justify-content-between">
                                    <strong class="fs-5">{{ selectedUF === 'Todos' ? '12,5 anos' : '11,8 anos' }}</strong>
                                </div>
                                <div class="progress flex-grow-1" style="height: 4px;"><div class="progress-bar bg-secondary" style="width: 100%"></div></div>
                                <small class="text-muted" style="font-size: 10px;">Idade Média da Frota</small>
                            </div>
                        </div>
                    </div>
                </div>

                <button @click="viewMode = 'detailed'" class="btn btn-detail w-100 py-3 mt-4 fw-bold shadow-sm">Avançar para Análise Detalhada</button>
            </div>
            </div>
        
        
      </div>
    </div>

    <!-- TELA 2: DETALHADA (FIEL AO MODELO SOLICITADO) -->
    <div v-show="viewMode === 'detailed'" class="view-wrapper detailed-bg pb-3 h-100 overflow-auto">
        <!-- Header Principal Tela 2 -->
        <div class="header-detailed bg-white border-bottom p-2 px-4 d-flex align-items-center justify-content-between sticky-top">
            <div class="d-flex align-items-center gap-3">
                <button @click="viewMode = 'executive'" class="btn btn-sm btn-light border rounded-circle"><ArrowLeft :size="18"/></button>
                <div class="d-flex align-items-center gap-2">
                    <span class="fw-bold text-dark">ESTADO:</span>
                    <select v-model="selectedUF" class="form-select border-0 fw-bold text-orange p-0 bg-transparent fs-7" style="width: auto;">
                        <option value="Todos">Brasil</option>
                        <option v-for="uf in ufs" :key="uf" :value="uf">{{ stateNames[uf] || uf }}</option>
                    </select>
                </div>
                <button @click="resetToBrazil" class="btn btn-sm btn-orange text-white fw-bold px-3 ms-3">Visualizar Brasil Todo</button>
            </div>
            <!-- <a href="#" @click.prevent="resetToBrazil" class="text-dark small text-decoration-underline">Aterar visualização para todo Brasil</a> -->
        </div>

        <div class="container-fluid px-4 mt-2">
            <!-- Barra de Filtros conforme Print -->
            <div class="card border-0 shadow-sm p-3 mb-3 bg-white rounded-3">
                <div class="row g-2 mb-2">
                    <div v-for="f in ['TIPO DE VEÍCULO', 'MARCA', 'VEÍCULO', 'MODELO', 'ANO', 'MOTOR', 'COMBUSTÍVEL']" :key="f" class="col">
                        <label class="filter-label">{{ f }}:</label>
                        <select class="form-select form-select-sm border-secondary-subtle"><option>Todos</option></select>
                    </div>
                </div>
                <div class="row g-2 align-items-end">
                    <div v-for="f in ['VISCOSIDADE', 'API', 'ACEA', 'JASO', 'NORMA', 'BÁSICO']" :key="f" class="col">
                        <label class="filter-label">{{ f }}:</label>
                        <select class="form-select form-select-sm border-secondary-subtle"><option>Todos</option></select>
                    </div>
                    <div class="col-auto">
                        <button class="btn btn-red-clear text-white fw-bold px-4">Limpar todos os filtros</button>
                    </div>
                </div>
            </div>

            <!-- KPIs com Volante conforme Print -->
            <!-- KPIs com Volante -->
            <div class="row g-3 mb-4">
                <div v-for="k in [
                    {t:'Marcas', v:'55', i:'v', clickable: true}, 
                    {t:'Modelos', v:'2.015', i:'v', clickable: true}, 
                    {t:'Veículos', v:'10.826.402', i:'v', clickable: false}, 
                    {t:'Litros / Ano', v:'71.583.150', i:'o', clickable: false}
                ]" :key="k.t" class="col-md-3">
                    <!-- Adicionamos o check k.clickable no @click -->
                    <div class="card border-0 shadow-sm p-4 rounded-4 card-white-kpi d-flex flex-row align-items-center justify-content-center gap-4" 
                        :class="{'cursor-default': !k.clickable}"
                        @click="k.clickable ? (showModal=true, modalTitle=k.t) : null">
                        
                        <div class="icon-circle-print flex-shrink-0" :style="!k.clickable ? 'border-color: #ccc' : ''">
                            <svg v-if="k.i === 'v'" width="36" height="36" viewBox="0 0 24 24" fill="none" :stroke="k.clickable ? '#e97332' : '#ccc'" stroke-width="2.5">
                                <circle cx="12" cy="12" r="10"/><circle cx="12" cy="12" r="3"/><path d="M12 15v5M5 12h3M16 12h3"/>
                            </svg>
                            <!-- Ícone de balde/óleo -->
                            <svg v-else width="36" height="36" viewBox="0 0 24 24" fill="none" stroke="#e97332" stroke-width="2.5">
                                <path d="M12 2l3 3h-6l3-3zM5 10h14v10a2 2 0 01-2 2H7a2 2 0 01-2-2V10z"/><path d="M9 14h6"/>
                            </svg>
                        </div>

                        <div class="text-start">
                            <h2 class="fw-bold m-0 text-dark" style="font-size: 1.8rem;">{{ k.v }}</h2>
                            <small class="text-muted fw-bold d-block" style="font-size: 0.9rem;">{{ k.t }}</small>
                        </div>
                    </div>
                </div>
            </div>

            <!-- Meio: Segmentação e Estados -->
            <div class="row g-2 mb-3">
                <div class="col-lg-5">
                    <div class="card border-0 shadow-sm p-3 h-100 rounded-4 bg-white">
                        <div class="row h-100 align-items-center">
                            <div v-for="(s, idx) in [
                                {l:'Leve', v:'6.896.270', lit:'26.121.998,68', i:Car, tr:'1,0'}, 
                                {l:'Pesada', v:'487.256', lit:'34.578.217,80', i:Truck, tr:'3,8'}, 
                                {l:'Moto', v:'3.442.876', lit:'10.882.933,17', i:Bike, tr:'3,0'}
                            ]" :key="s.l" class="col-4 border-end last-no-border px-3">
                                
                                <!-- Ícone e Veículos (Lado a Lado) -->
                                <div class="d-flex align-items-center gap-2 mb-3">
                                    <component :is="s.i" class="text-orange flex-shrink-0" :size="42" />
                                    <div class="lh-sm">
                                        <strong class="fs-6 d-block">{{ s.v }}</strong>
                                        <small class="text-muted fw-bold" style="font-size: 11px;">Veículos</small>
                                    </div>
                                </div>

                                <!-- Litros / Ano -->
                                <div class="mb-3 border-top pt-2">
                                    <strong class="fs-6 d-block">{{ s.lit }}</strong>
                                    <small class="text-muted fw-bold" style="font-size: 11px;">Litros / ano</small>
                                </div>

                                <!-- Box de Trocas por Ano -->
                                <div class="trocas-print-container p-2 rounded-3 border-light-subtle border">
                                    <small class="text-muted fw-bold d-block mb-2 text-center" style="font-size: 11px;">Trocas por Ano</small>
                                    <div class="d-flex align-items-center justify-content-center gap-2">
                                        <input type="text" class="form-control form-control-sm text-center fw-bold bg-white" style="width: 55px; height: 30px;" :value="s.tr">
                                        <Info :size="16" class="text-muted" />
                                    </div>
                                </div>
                            </div>
                        </div>
                        
                        <!-- Rodapé do Card -->
                        <div class="mt-3 d-flex gap-2 align-items-center">
                            <Info :size="16" class="text-muted" />
                            <small class="text-muted" style="font-size: 10px;">
                                A quantidade de trocas apresentada, representa uma estimativa baseada na média nacional.
                            </small>
                        </div>
                    </div>
                </div>
                <!-- CARD ESTADOS - VERSÃO COM EIXOS E GRADES CONFORME O PRINT -->
                <div class="col-lg-4">
                    <div class="card border-0 shadow-sm p-3 h-100 rounded-4 bg-white">
                        <!-- Título alinhado à esquerda -->
                        <small class="fw-bold text-muted uppercase d-block mb-2 text-start" style="font-size: 11px;">ESTADOS</small>
                        
                        <div class="flex-grow-1" style="height: 150px;">
                            <Bar 
                                :data="{
                                    labels: ['MG'],
                                    datasets: [{
                                        backgroundColor: '#e97332',
                                        data: [71.5],
                                        barThickness: 80
                                    }]
                                }" 
                                :options="{ 
                                    responsive: true, 
                                    maintainAspectRatio: false, 
                                    plugins: { legend: { display: false } },
                                    scales: { 
                                        y: { 
                                            beginAtZero: true,
                                            min: 0,
                                            max: 100,
                                            grid: { 
                                                display: true,
                                                borderDash: [3, 3], // Linha pontilhada/tracejada
                                                color: '#e2e8f0',
                                                drawBorder: false,
                                                drawTicks: false
                                            },
                                            ticks: { 
                                                font: { size: 10 },
                                                stepSize: 50, // Garante os rótulos 0 Mi e 50 Mi
                                                callback: value => value + ' Mi'
                                            }
                                        }, 
                                        x: { 
                                            grid: { display: false },
                                            ticks: { 
                                                font: { size: 10, weight: 'bold' }, 
                                                color: '#333' 
                                            }
                                        } 
                                    } 
                                }" 
                            />
                        </div>
                    </div>
                </div>
                <!-- CARD TOP CIDADES - VERSÃO FINAL COM NOMES E GRADES VERTICAIS -->
                <div class="col-lg-3">
                    <div class="card border-0 shadow-sm p-3 h-100 rounded-4 bg-white">
                        <!-- Título alinhado à esquerda -->
                        <small class="fw-bold text-muted uppercase d-block mb-4 text-start" style="font-size: 11px;">TOP CIDADES</small>
                        
                        <div class="flex-grow-1" style="height: 180px;">
                            <Bar 
                                :data="{
                                    labels: ['BELO HORIZONTE...', 'UBERLANDIA', 'CONTAGEM', 'JUIZ DE FORA', 'MONTES CLAROS'],
                                    datasets: [{
                                        label: 'Litros',
                                        backgroundColor: '#e97332',
                                        data: [14.5, 4.2, 3.8, 2.5, 2.1],
                                        barThickness: 15
                                    }]
                                }" 
                                :options="{ 
                                    indexAxis: 'y', // Define o gráfico como horizontal
                                    responsive: true, 
                                    maintainAspectRatio: false, 
                                    plugins: { legend: { display: false } },
                                    scales: { 
                                        x: { 
                                            beginAtZero: true,
                                            grid: { 
                                                display: true,
                                                borderDash: [3, 3], // Linha tracejada vertical
                                                color: '#e2e8f0',
                                                drawBorder: false
                                            },
                                            ticks: { 
                                                font: { size: 10 },
                                                stepSize: 10,
                                                callback: value => value + ' Mi' // Sufixo Mi no eixo X
                                            }
                                        }, 
                                        y: { 
                                            grid: { display: false },
                                            ticks: { 
                                                font: { size: 9, weight: 'bold' }, 
                                                color: '#666'
                                            }
                                        } 
                                    } 
                                }" 
                            />
                        </div>
                    </div>
                </div>
            </div>

            <!-- Gráficos Inferiores Ajustados para ficarem na mesma linha -->
            <div class="row g-2">
                
                <!-- CARD VISCOSIDADE (Largo: 4/12) -->
                <div class="col-md-5">
                    <div class="card border-0 shadow-sm p-3 rounded-4 h-100 bg-white">
                        <div class="d-flex justify-content-between align-items-center mb-3">
                            <small class="fw-bold text-muted uppercase text-start" style="font-size: 11px;">VISCOSIDADE</small>
                            <!-- Legenda customizada estilo print -->
                            <div class="d-flex gap-2" style="font-size: 9px;">
                                <span class="d-flex align-items-center gap-1"><span class="dot" style="background:#3b82f6"></span> Volume</span>
                                <span class="d-flex align-items-center gap-1"><span class="dot" style="background:#e97332"></span> % Acumulado</span>
                            </div>
                        </div>
                        
                        <div class="flex-grow-1 overflow-x-auto pb-2 scroll-custom">
                            <div style="height: 180px; min-width: 550px;">
                                <!-- Usamos o componente Bar, mas com datasets mistos -->
                                <Bar 
                                    :data="{
                                        labels: ['15W40', '5W30', '10W40', '10W30', '5W40', '20W50', '0W20', '15W50', '0W30', '5W20', '10W50', '0W40'],
                                        datasets: [
                                            {
                                                type: 'line',
                                                label: '% Acumulado',
                                                borderColor: '#e97332',
                                                backgroundColor: '#e97332',
                                                borderWidth: 2,
                                                data: [41, 61, 73, 81, 89, 93, 97, 98, 99, 99.5, 99.8, 100], // Valores calculados do acumulado
                                                yAxisID: 'y1',
                                                tension: 0.1,
                                                pointRadius: 4,
                                                pointBackgroundColor: '#e97332',
                                                datalabels: {
                                                    align: 'top',
                                                    formatter: (v) => v + '%'
                                                }
                                            },
                                            {
                                                type: 'bar',
                                                label: 'Litros',
                                                backgroundColor: '#3b82f6', // Azul conforme o print modelo
                                                data: [29.7, 14.3, 8.6, 6.0, 5.1, 3.5, 2.2, 0.7, 0.7, 0.7, 0.1, 0.1],
                                                yAxisID: 'y',
                                                barThickness: 30,
                                                datalabels: {
                                                    anchor: 'end',
                                                    align: 'top',
                                                    formatter: (val) => val.toFixed(1).replace('.', ',')
                                                }
                                            }
                                        ]
                                    }" 
                                    :options="{
                                        responsive: true,
                                        maintainAspectRatio: false,
                                        layout: { padding: { top: 30, right: 30 } },
                                        plugins: {
                                            legend: { display: false },
                                            datalabels: {
                                                color: '#444',
                                                font: { size: 10, weight: 'bold' }
                                            },
                                            tooltip: {
                                                enabled: true,
                                                backgroundColor: '#fff',
                                                titleColor: '#333',
                                                bodyColor: '#666',
                                                borderColor: '#ddd',
                                                borderWidth: 1,
                                                padding: 10,
                                                cornerRadius: 4,
                                                callbacks: {
                                                    title: (items) => `Viscosidade: ${items[0].label}`,
                                                    label: (item) => item.dataset.type === 'bar' 
                                                        ? `Volume: ${item.raw} Mi Litros` 
                                                        : `Acumulado: ${item.raw}%`
                                                }
                                            }
                                        },
                                        
                                        scales: {
                                                y: { 
                                                    type: 'linear',
                                                    display: true,
                                                    position: 'left',
                                                    // Alteração: Definimos um max maior que o maior valor (29.7) 
                                                    // para sobrar espaço no topo das barras
                                                    min: 0,
                                                    max: 55, 
                                                    grid: { display: false },
                                                    ticks: { font: { size: 9 } }
                                                },
                                                y1: {
                                                    type: 'linear',
                                                    display: true,
                                                    position: 'right',
                                                    min: -40, 
                                                    max: 110,
                                                    grid: { 
                                                        drawOnChartArea: true,
                                                        borderDash: [3, 3],
                                                        color: '#e2e8f0'
                                                    },
                                                    ticks: { 
                                                        font: { size: 9 },
                                                        callback: (val) => val >= 0 ? val + '%' : '' // Esconde valores negativos na legenda
                                                    }
                                                },
                                                x: { 
                                                    grid: { display: false },
                                                    ticks: { font: { size: 9, weight: 'bold' } }
                                                }
                                            }
                                    }" 
                                />
                            </div>
                        </div>
                    </div>
                </div>

                <!-- CARD BÁSICO (Estreito: 2/12) -->
                <div class="col-md-2">
                    <div class="card border-0 shadow-sm p-3 rounded-4 h-100 bg-white d-flex flex-column">
                        <small class="fw-bold text-muted mb-2 d-block text-start" style="font-size: 11px;">BÁSICO</small>
                        <div class="flex-grow-1" style="height: 150px;">
                            <Doughnut :data="basicData" :options="doughnutOptionsExtended" />
                        </div>
                        <div class="mt-2 d-flex align-items-center justify-content-center gap-1 flex-wrap" style="font-size: 8.5px;">
                            <span class="fw-bold text-muted uppercase">LUBRIFICA...</span>
                            <span class="d-flex align-items-center gap-1"><span class="dot" style="background:#e97332"></span> MINERAL</span>
                            <span class="d-flex align-items-center gap-1"><span class="dot" style="background:#f4b393"></span> SINTÉTICO</span>
                        </div>
                    </div>
                </div>

                <!-- CARD SEGMENTO (Estreito: 2/12) -->
                <div class="col-md-2">
                    <div class="card border-0 shadow-sm p-3 rounded-4 h-100 bg-white d-flex flex-column">
                        <small class="fw-bold text-muted mb-2 d-block text-start" style="font-size: 11px;">SEGMENTO</small>
                        <div class="flex-grow-1" style="height: 150px;">
                            <Doughnut :data="segmentoData" :options="doughnutOptionsExtended" />
                        </div>
                        <div class="mt-2 d-flex align-items-center justify-content-center gap-1" style="font-size: 8.5px;">
                            <span class="fw-bold text-muted uppercase">TIPO</span>
                            <span class="d-flex align-items-center gap-1"><span class="dot" style="background:#e97332"></span> LEVE</span>
                            <span class="d-flex align-items-center gap-1"><span class="dot" style="background:#f4b393"></span> MOTO</span>
                        </div>
                    </div>
                </div>

                <!-- CARD NORMAS (Médio: 4/12) -->
                <div class="col-md-3">
                    <div class="card border-0 shadow-sm p-3 rounded-4 h-100 bg-white">
                        <small class="fw-bold text-muted mb-2 d-block text-start" style="font-size: 11px;">NORMAS</small>
                        <div style="height: 180px;">
                            <Doughnut :data="normasData" :options="normasOptions" /> <!-- Alterado aqui -->
                        </div>
                    </div>
                </div>

            </div>
        </div>
    </div>

    <!-- MODAL DE PIZZA ATUALIZADO -->
    <div v-if="showModal" class="modal-overlay d-flex align-items-center justify-content-center" @click.self="showModal = false">
        <div class="modal-content bg-white p-5 rounded-4 shadow-lg position-relative" style="width: 550px;">
            <button @click="showModal = false" class="btn-close-custom"><X :size="20"/></button>
            
            <div class="text-center mb-4">
                <h4 class="fw-bold text-dark m-0">Volume por {{ modalTitle }}</h4>
                <small class="text-orange fw-bold">Filtro aplicado: Viscosidade Selecionada</small>
            </div>

            <div style="height: 350px;">
                <!-- Usando modalChartData que alterna entre Marcas e Modelos -->
                <Pie :data="modalChartData" :options="{
                    responsive: true,
                    maintainAspectRatio: false,
                    plugins: {
                        legend: { position: 'bottom' },
                        datalabels: {
                            color: '#fff',
                            font: { weight: 'bold' },
                            formatter: (value) => value + '%'
                        }
                    }
                }"/>
            </div>
            
            <div class="mt-4 p-3 bg-light rounded-3">
                <small class="text-muted d-block text-center">
                    Este gráfico exibe a participação das principais <strong>{{ modalTitle.toLowerCase() }}</strong> 
                    dentro do volume total da viscosidade filtrada.
                </small>
            </div>
        </div>
    </div>

  </div>
</template>

<style scoped>
.dashboard-container { background-color: #f1f5f9; min-height: 100vh; font-family: 'Inter', sans-serif; overflow: hidden; }
.detailed-bg { background-color: #f8fafc; }
.text-orange { color: #e97332; }
.bg-orange { background-color: #e97332; }
.uppercase { text-transform: uppercase; letter-spacing: 0.8px; }

/* Dashboard Executive */
.icon-header { width: 48px; height: 48px; background-color: #e97332; border-radius: 12px; display: flex; align-items: center; justify-content: center; box-shadow: 0 4px 10px rgba(233, 115, 50, 0.3); }
.info-alert { background-color: #f1f5f9; border: 1px solid #e2e8f0; font-size: 11px; }
.mini-insight-icon { width: 26px; height: 26px; background: #fff1eb; border-radius: 6px; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
.btn-detail { background-color: #eee; color: #888; border: none; transition: 0.3s; }
.btn-detail:hover { background-color: #e97332; color: white; transform: translateY(-2px); }

/* Dashboard Detailed - Estilo Print */
.btn-orange { background-color: #e97332; transition: 0.2s; }
.btn-orange:hover { background-color: #d15f22; }
.filter-label { font-size: 10px; font-weight: 800; color: #333; margin-bottom: 2px; display: block; }
.form-select-sm { font-size: 11px; font-weight: 600; border-radius: 6px; }
.btn-red-clear { background-color: #ff0000; font-size: 12px; border: none; border-radius: 8px; transition: 0.2s; height: 32px; }
.btn-red-clear:hover { background-color: #cc0000; }

.card-white-kpi { cursor: pointer; border: 1px solid transparent; transition: 0.2s; }
.card-white-kpi:hover { border-color: #e97332; background: #fffdfc; }
.icon-circle-print { width: 50px; height: 50px; border-radius: 50%; border: 2.5px solid #e97332; display: flex; align-items: center; justify-content: center; }

.trocas-print-box { border: 1px solid #dee2e6; border-radius: 8px; padding: 10px; background: #fdfdfd; }
.last-no-border:last-child { border-right: none !important; }

/* Modal */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.6); z-index: 9999; backdrop-filter: blur(4px); }
.btn-close-custom { position: absolute; top: 15px; right: 15px; border: none; background: #eee; border-radius: 50%; width: 30px; height: 30px; }

/* Geral */
.progress { background-color: #eee; border-radius: 10px; }
.custom-hint { position: absolute; background: white; border-radius: 8px; z-index: 1000; pointer-events: none; min-width: 170px; font-size: 0.8rem; }

@keyframes fadeIn { from { opacity: 0; transform: translateY(8px); } to { opacity: 1; transform: translateY(0); } }
.view-wrapper { animation: fadeIn 0.4s ease-out; }
:deep(.mapboxgl-ctrl-bottom-right), :deep(.mapboxgl-ctrl-bottom-left) { display: none; }

.dot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    display: inline-block;
}

/* Garante que o texto da legenda não quebre feio */
.uppercase {
    text-transform: uppercase;
    letter-spacing: 0.5px;
}

/* Ajuste para o layout dos cards */
.card {
    transition: transform 0.2s;
}

cursor-default {
    cursor: default !important;
}

.card-white-kpi.cursor-default:hover {
    border-color: transparent !important;
    background: white !important;
}

</style>
