<script setup>
import { ref, computed, onMounted, watch, reactive, nextTick } from 'vue';
import axios from 'axios';
import { Bar, Doughnut, Pie } from 'vue-chartjs';
import { 
  Chart as ChartJS, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale, RadialLinearScale, ArcElement, PointElement, LineElement, BarController, LineController
} from 'chart.js';
import ChartDataLabels from 'chartjs-plugin-datalabels';
import { 
  Car, Bike, Truck, Info, X, Clock, Zap, Map as MapIcon, ArrowLeft
} from 'lucide-vue-next'; 
import mapboxgl from 'mapbox-gl';
import 'mapbox-gl/dist/mapbox-gl.css';

// --- CONFIGURAÇÃO API ---
const api = axios.create({ baseURL: 'http://localhost:3000/bi/regiao-2', timeout: 20000 });

ChartJS.register(
    CategoryScale, LinearScale, RadialLinearScale, BarElement, ArcElement, 
    PointElement, LineElement, BarController, LineController, Title, Tooltip, Legend, ChartDataLabels
);

// --- ESTADOS ---
const viewMode = ref('executive'); 
const selectedUF = ref('Todos');
const selectedStateName = ref('Brasil');
const isLoading = ref(false);
const isModalLoading = ref(false);
const showModal = ref(false);
const modalTitle = ref('');
const modalData = ref(null);
const modalChartType = ref('pie'); // 'pie' para marcas/modelos, 'bar' para estados completo

const apiFilters = ref(null);
const filters = reactive({
    tipoVeiculo: 'Todos', marca: 'Todos', veiculo: 'Todos', modelo: 'Todos',
    ano: 'Todos', motor: 'Todos', combustivel: 'Todos', viscosidade: 'Todos',
    api: 'Todos', acea: 'Todos', jaso: 'Todos', norma: 'Todos', basico: 'Todos',
    // Fatores de troca
    trocaLeve: 1, 
    trocaPesada: 3.8, 
    trocaMoto: 3
});

const panorama = ref(null);
const detalhada = ref(null);

const stateNames = {
  'AC': 'Acre', 'AL': 'Alagoas', 'AP': 'Amapá', 'AM': 'Amazonas', 'BA': 'Bahia', 'CE': 'Ceará', 'DF': 'Distrito Federal', 'ES': 'Espírito Santo', 'GO': 'Goiás', 'MA': 'Maranhão', 'MT': 'Mato Grosso', 'MS': 'Mato Grosso do Sul', 'MG': 'Minas Gerais', 'PA': 'Pará', 'PB': 'Paraíba', 'PR': 'Paraná', 'PE': 'Pernambuco', 'PI': 'Piauí', 'RJ': 'Rio de Janeiro', 'RN': 'Rio Grande do Norte', 'RS': 'Rio Grande do Sul', 'RO': 'Rondônia', 'RR': 'Roraima', 'SC': 'Santa Catarina', 'SP': 'São Paulo', 'SE': 'Sergipe', 'TO': 'Tocantins'
};

// --- FUNÇÕES DE API ---

const loadData = async () => {
    if (isLoading.value) return;
    isLoading.value = true;
    const ufParam = selectedUF.value === 'Todos' ? '' : selectedUF.value;
    
    try {
        const params = { estado: ufParam, ...filters };

        if (viewMode.value === 'executive') {
            const res = await api.get('/panorama', { params });
            panorama.value = res.data.data;
        } else {
            const res = await api.get('/analise-detalhada', { params });
            detalhada.value = res.data.data;
            
            // PROTEÇÃO: Garante que os fatores de troca não fiquem vazios (Fix do 3.8)
            if (detalhada.value.parametros?.defaults && !apiFilters.value) {
                filters.trocaLeve = Number(detalhada.value.parametros.defaults.trocaLeve) || 1;
                filters.trocaPesada = Number(detalhada.value.parametros.defaults.trocaPesada) || 3.8;
                filters.trocaMoto = Number(detalhada.value.parametros.defaults.trocaMoto) || 3;
                const fRes = await api.get('/filtros');
                apiFilters.value = fRes.data.data;
            }
        }
    } catch (e) { console.error("Erro API:", e); } finally {
        isLoading.value = false;
        if (map.value) nextTick(() => map.value.resize());
    }
};

const openDetailedModal = async (type) => {
    modalTitle.value = type;
    modalChartType.value = 'pie'; // Marcas e Modelos usam Pizza
    showModal.value = true;
    modalData.value = null;
    isModalLoading.value = true;
    const ufParam = selectedUF.value === 'Todos' ? '' : selectedUF.value;
    try {
        const res = await api.get(type === 'Marcas' ? '/graficos/marcas' : '/graficos/modelos', { 
            params: { estado: ufParam, ...filters } 
        });
        modalData.value = res.data.data;
    } catch (e) { console.error(e); } finally { isModalLoading.value = false; }
};

// Nova função exclusiva para abrir o modal de estados largo
const openStatesModal = () => {
    modalTitle.value = 'Volume por Estado (Completo)';
    modalChartType.value = 'bar'; // Define que o modal será o de barras largo
    showModal.value = true;
    modalData.value = true; 
};

const clearFilters = () => {
    Object.keys(filters).forEach(k => { if (!k.startsWith('troca')) filters[k] = 'Todos'; });
    loadData();
};

// --- MAPBOX ---
const mapContainer = ref(null);
const map = ref(null);
const MAPBOX_TOKEN = import.meta.env.VITE_MAPBOX_TOKEN || 'SUA_CHAVE_AQUI';

const initMap = () => {
    mapboxgl.accessToken = MAPBOX_TOKEN;
    map.value = new mapboxgl.Map({ 
        container: mapContainer.value, style: 'mapbox://styles/mapbox/light-v11', 
        center: [-52, -15], zoom: 3.5, projection: 'mercator', attributionControl: false 
    });
    map.value.on('load', () => {
        map.value.addSource('brazil-states', { type: 'geojson', data: 'https://raw.githubusercontent.com/codeforamerica/click_that_hood/master/public/data/brazil-states.geojson' });
        map.value.addLayer({ 'id': 'states-fill', 'type': 'fill', 'source': 'brazil-states', 'paint': { 'fill-color': '#f97316', 'fill-opacity': 0.1 }});
        map.value.addLayer({ 'id': 'states-borders', 'type': 'line', 'source': 'brazil-states', 'paint': { 'line-color': '#e97332', 'line-width': 0.5 }});
        map.value.on('click', 'states-fill', (e) => { selectedUF.value = e.features[0].properties.sigla; });
        updateMapHighlight();
    });
};

const updateMapHighlight = () => {
    if(!map.value || !map.value.isStyleLoaded()) return;
    map.value.setPaintProperty('states-fill', 'fill-color', ['case', ['==', ['get', 'sigla'], selectedUF.value], '#e97332', '#f97316']);
    map.value.setPaintProperty('states-fill', 'fill-opacity', ['case', ['==', ['get', 'sigla'], selectedUF.value], 0.5, 0.1]);
};

// --- GRÁFICOS ---
const orangePalette = ['#e97332', '#f4b393', '#fbe2d5', '#3b82f6', '#10b981', '#f59e0b', '#00bcd4'];

const chartViscosidade = computed(() => {
    if (!detalhada.value?.graficos?.viscosidade) return { labels: [], datasets: [] };
    const items = detalhada.value.graficos.viscosidade;
    let acc = 0;
    const total = items.reduce((a, b) => a + b.litros, 0);
    const acumulado = items.map(i => { acc += i.litros; return ((acc / total) * 100).toFixed(1); });

    return {
        labels: items.map(i => i.label),
        datasets: [
            { 
                type: 'line', label: '% Acumulado', borderColor: '#e97332', data: acumulado, yAxisID: 'y1', tension: 0.2, pointRadius: 4, pointBackgroundColor: '#e97332',
                datalabels: { display: true, align: 'top', color: '#e97332', font: { weight: 'bold', size: 10 }, formatter: v => v + '%' }
            },
            { 
                type: 'bar', label: 'Volume Mi', backgroundColor: '#3b82f6', data: items.map(i => i.litros / 1000000), yAxisID: 'y', barThickness: 28,
                datalabels: { display: true, anchor: 'end', align: 'top', color: '#444', font: { weight: 'bold', size: 10 }, formatter: v => v.toFixed(1) }
            }
        ]
    };
});

// Gráfico Card: Apenas Top 10
const chartEstados = computed(() => {
    if (!detalhada.value?.graficos?.estado) return { labels: [], datasets: [] };
    const items = detalhada.value.graficos.estado.slice(0, 10);
    return {
        labels: items.map(e => e.estado),
        datasets: [{ 
            backgroundColor: '#e97332', 
            data: items.map(e => e.litrosAno / 1000000), 
            barThickness: 25,
            datalabels: { anchor: 'end', align: 'top', formatter: v => v.toFixed(1), font: { size: 9, weight: 'bold' } }
        }]
    };
});

// Gráfico Modal: Todos os Estados
const chartEstadosFull = computed(() => {
    if (!detalhada.value?.graficos?.estado) return { labels: [], datasets: [] };
    const items = detalhada.value.graficos.estado;
    return {
        labels: items.map(e => e.estado),
        datasets: [{ 
            backgroundColor: '#e97332', 
            data: items.map(e => e.litrosAno / 1000000), 
            barThickness: 15,
            datalabels: { anchor: 'end', align: 'top', formatter: v => v.toFixed(1), font: { size: 9, weight: 'bold' } }
        }]
    };
});

// --- NOVAS COMPUTEDS PARA CIDADES ---

// Gráfico do Card: Apenas Top 5 (Horizontal)
const chartCidadesTop5 = computed(() => {
    if (!detalhada.value?.graficos?.topCidades) return { labels: [], datasets: [] };
    const items = detalhada.value.graficos.topCidades.slice(0, 5);
    return {
        labels: items.map(c => c.cidade),
        datasets: [{ 
            backgroundColor: '#e97332', 
            data: items.map(c => (c.litrosAno / 1000000)), 
            barThickness: 15,
            datalabels: { anchor: 'end', align: 'right', formatter: v => v.toFixed(2), font: { size: 9, weight: 'bold' } }
        }]
    };
});

// Gráfico do Modal: Top 30 (Vertical)
const chartCidadesFull = computed(() => {
    if (!detalhada.value?.graficos?.topCidades) return { labels: [], datasets: [] };
    const items = detalhada.value.graficos.topCidades.slice(0, 30);
    return {
        labels: items.map(c => c.cidade),
        datasets: [{ 
            backgroundColor: '#e97332', 
            data: items.map(c => (c.litrosAno / 1000000)), 
            barThickness: 12,
            datalabels: { anchor: 'end', align: 'top', formatter: v => v.toFixed(2), font: { size: 8 } }
        }]
    };
});

// Função para abrir o modal de Cidades
const openCitiesModal = () => {
    modalTitle.value = 'Top 30 Cidades (Volume Mi L)';
    modalChartType.value = 'bar-cities'; // Tipo exclusivo para cidades
    showModal.value = true;
    modalData.value = true;
};

const doughnutOptionsStandard = {
    responsive: true, maintainAspectRatio: false,
    plugins: {
        legend: { display: false },
        datalabels: {
            color: '#444', font: { size: 10, weight: 'bold' },
            formatter: (v, ctx) => {
                const total = ctx.dataset.data.reduce((a, b) => a + b, 0);
                return total > 0 ? ((v / total) * 100).toFixed(1) + '%' : '0%';
            }
        }
    },
    cutout: '60%'
};

// --- WATCHERS ---
watch([selectedUF, viewMode], () => { 
    if (selectedUF.value === 'Todos') selectedStateName.value = 'Brasil';
    else selectedStateName.value = stateNames[selectedUF.value];
    updateMapHighlight(); 
    loadData(); 
});
watch(filters, () => { loadData(); }, { deep: true });
onMounted(() => { initMap(); loadData(); });
</script>

<template>
  <div class="dashboard-container">
    
    <!-- TELA 1: PANORAMA -->
    <div v-show="viewMode === 'executive'" class="view-wrapper p-4">
      <div class="row g-4 align-items-stretch">
        <div class="col-lg-7">
          <div class="card map-card position-relative border-0 shadow-sm rounded-4 overflow-hidden bg-white" style="min-height: 750px;">
            <div class="p-4 position-absolute top-0 start-0 z-2">
              <h3 class="fw-bold m-0" style="color: #e97332">Panorama <span class="text-dark">Estratégico por Região</span></h3>
              <p class="text-muted small">Selecione no mapa ou clique <a href="#" @click.prevent="selectedUF = 'Todos'" class="text-primary">AQUI</a> para o Brasil</p>
            </div>
            <div ref="mapContainer" class="position-absolute top-0 start-0 w-100 h-100"></div>
            <div v-if="isLoading" class="loader-overlay"><div class="spinner-border text-orange"></div></div>
          </div>
        </div>

        <div class="col-lg-5">
            <div class="card h-100 border-0 shadow-sm p-4 d-flex flex-column rounded-4 bg-white">
                <div class="d-flex align-items-center gap-3 mb-4">
                    <div class="icon-header"><MapIcon class="text-white" :size="24"/></div>
                    <h2 class="fw-bold m-0">{{ selectedStateName }}</h2>
                </div>
                <div v-if="panorama">
                    <div class="row mb-4">
                        <div class="col-6"><h3 class="fw-bold m-0">{{ new Intl.NumberFormat('pt-BR').format(panorama.resumo.litrosAnoTotal) }}</h3><small class="text-muted">Litros / ano total</small></div>
                        <div class="col-6 border-start ps-4"><h3 class="fw-bold m-0">{{ new Intl.NumberFormat('pt-BR').format(panorama.resumo.frotaGeral) }}</h3><small class="text-muted">Frota Geral</small></div>
                    </div>
                    <hr class="mb-4">
                    <div class="flex-grow-1">
                        <label class="fw-bold mb-4 d-block">Segmentação e Perfil:</label>
                        <div v-for="s in panorama.perfilSegmentacao" :key="s.id" class="mb-4">
                            <div class="d-flex align-items-start gap-3">
                                <component :is="s.id.includes('leve')?Car:s.id.includes('pesada')?Truck:Bike" class="text-orange mt-1" :size="32"/>
                                <div class="flex-grow-1">
                                    <div class="d-flex justify-content-between">
                                        <strong>{{ new Intl.NumberFormat('pt-BR').format(s.litrosAno) }} L</strong>
                                        <small class="text-muted">{{ new Intl.NumberFormat('pt-BR').format(s.frota) }} veíc.</small>
                                    </div>
                                    <div class="progress" style="height: 6px;"><div class="progress-bar bg-orange" :style="{width: s.percentualLitros + '%'}"></div></div>
                                    <small class="text-muted" style="font-size: 10px;">{{ s.label }} ({{ s.percentualLitros }}%)</small>
                                </div>
                            </div>
                        </div>
                        <div class="pt-3 border-top d-flex align-items-center gap-3">
                            <Clock class="text-orange" :size="32"/>
                            <div><strong class="fs-5">{{ panorama.resumo.idadeMediaFrota }} anos</strong><small class="text-muted d-block" style="font-size:10px">Idade Média da Frota</small></div>
                        </div>
                    </div>
                </div>
                <button @click="viewMode = 'detailed'" class="btn btn-detail w-100 py-3 mt-4 fw-bold shadow-sm">Avançar para Análise Detalhada</button>
            </div>
        </div>
      </div>
    </div>

    <!-- TELA 2: DETALHADA -->
    <div v-show="viewMode === 'detailed'" class="view-wrapper detailed-bg pb-3 h-100 overflow-auto">
        <div class="header-detailed bg-white border-bottom p-2 px-4 d-flex align-items-center justify-content-between sticky-top">
            <div class="d-flex align-items-center gap-3">
                <button @click="viewMode = 'executive'" class="btn btn-sm btn-light border rounded-circle"><ArrowLeft :size="18"/></button>
                <div class="d-flex align-items-center gap-2">
                    <span class="fw-bold text-dark">ESTADO:</span>
                    <select v-model="selectedUF" class="form-select border-0 fw-bold text-orange p-0 bg-transparent fs-7" style="width: auto;">
                        <option value="Todos">Brasil</option><option v-for="(name, sigla) in stateNames" :key="sigla" :value="sigla">{{ name }}</option>
                    </select>
                </div>
            </div>
            <div v-if="isLoading" class="spinner-border spinner-border-sm text-orange"></div>
        </div>

        <div class="container-fluid px-4 mt-2" v-if="detalhada">
            <!-- Barra de Filtros -->
            <div class="card border-0 shadow-sm p-3 mb-3 bg-white rounded-3">
                <div class="row g-2 mb-2" v-if="apiFilters">
                    <div v-for="f in [{l:'TIPO DE VEÍCULO', k:'tipoVeiculo'}, {l:'MARCA', k:'marca'}, {l:'VEÍCULO', k:'veiculo'}, {l:'MODELO', k:'modelo'}, {l:'ANO', k:'ano'}, {l:'MOTOR', k:'motor'}, {l:'COMBUSTÍVEL', k:'combustivel'}]" :key="f.k" class="col">
                        <label class="filter-label">{{ f.l }}:</label>
                        <select v-model="filters[f.k]" class="form-select form-select-sm border-secondary-subtle">
                            <option value="Todos">Todos</option><option v-for="o in apiFilters[f.k]" :key="o" :value="o">{{ o }}</option>
                        </select>
                    </div>
                </div>
                <div class="row g-2 align-items-end" v-if="apiFilters">
                    <div v-for="f in [{l:'VISCOSIDADE', k:'viscosidade'}, {l:'API', k:'api'}, {l:'ACEA', k:'acea'}, {l:'JASO', k:'jaso'}, {l:'NORMA', k:'norma'}, {l:'BÁSICO', k:'basico'}]" :key="f.k" class="col">
                        <label class="filter-label">{{ f.l }}:</label>
                        <select v-model="filters[f.k]" class="form-select form-select-sm border-secondary-subtle">
                            <option value="Todos">Todos</option><option v-for="o in apiFilters[f.k]" :key="o" :value="o">{{ o }}</option>
                        </select>
                    </div>
                    <div class="col-auto"><button @click="clearFilters" class="btn btn-red-clear text-white fw-bold px-4">Limpar Filtros</button></div>
                </div>
            </div>

            <!-- KPIs TOPO -->
            <div class="row g-3 mb-4">
                <div v-for="k in detalhada.cardsTopo" :key="k.id" class="col-md-3">
                    <div class="card border-0 shadow-sm p-4 rounded-4 bg-white d-flex flex-row align-items-center justify-content-center gap-3" 
                         :class="{'card-white-kpi': k.id === 'marcas' || k.id === 'modelos'}"
                         @click="(k.id === 'marcas' || k.id === 'modelos') ? openDetailedModal(k.label) : null">
                        <div class="icon-circle-print"><Zap v-if="k.id==='litros-ano'" class="text-orange" :size="24" /><Car v-else class="text-orange" :size="24" /></div>
                        <div><h2 class="fw-bold m-0 text-dark" style="font-size:1.7rem">{{ new Intl.NumberFormat('pt-BR').format(k.valor) }}</h2><small class="text-muted fw-bold d-block">{{ k.label }}</small></div>
                    </div>
                </div>
            </div>

            <!-- MEIO -->
            <div class="row g-2 mb-3">
                <div class="col-lg-5">
                    <div class="card border-0 shadow-sm p-3 h-100 rounded-4 bg-white">
                        <div class="row h-100 align-items-center">
                            <div v-for="s in detalhada.comparativosSegmento" :key="s.id" class="col-4 border-end last-no-border px-3">
                                <div class="d-flex align-items-center gap-2 mb-3">
                                    <component :is="s.id.includes('leve')?Car:s.id.includes('pesada')?Truck:Bike" class="text-orange" :size="42" />
                                    <div class="lh-sm"><strong>{{ new Intl.NumberFormat('pt-BR').format(s.veiculos) }}</strong><small class="text-muted d-block" style="font-size:10px">Veículos</small></div>
                                </div>
                                <div class="mb-3 border-top pt-2"><strong>{{ new Intl.NumberFormat('pt-BR').format(s.litrosAno) }}</strong><small class="text-muted d-block" style="font-size:10px">Litros / ano</small></div>
                                <div class="p-2 rounded-3 border border-light-subtle text-center bg-white shadow-sm">
                                    <small class="text-muted d-block mb-1 fw-bold" style="font-size:10px">Trocas por Ano</small>
                                    <input type="number" step="0.1" class="form-control form-control-sm text-center fw-bold bg-light border-0" 
                                           v-model.number="filters[s.id.includes('leve') ? 'trocaLeve' : (s.id.includes('pesada') ? 'trocaPesada' : 'trocaMoto')]">
                                </div>
                            </div>
                        </div>
                    </div>
                </div>

                <div class="col-lg-4">
                    <!-- Card de Estados (Top 10) com Clique Exclusivo -->
                    <div class="card border-0 shadow-sm p-3 h-100 rounded-4 bg-white card-white-kpi" @click="openStatesModal">
                        <div class="d-flex justify-content-between align-items-start mb-2">
                            <small class="fw-bold text-muted uppercase">VOL. POR ESTADO (Top 10)</small>
                            <Info :size="14" class="text-muted" /> 
                        </div>
                        <div style="height: 180px;"><Bar :data="chartEstados" :options="{ responsive: true, maintainAspectRatio: false, plugins: { legend: { display: false } }, scales: { x: { ticks: { font: { size: 9 } } } } }" /></div>
                    </div>
                </div>

                <div class="col-lg-3">
                    <!-- Adicionamos card-white-kpi e @click -->
                    <div class="card border-0 shadow-sm p-3 h-100 rounded-4 bg-white card-white-kpi" @click="openCitiesModal">
                        <div class="d-flex justify-content-between align-items-start mb-2">
                            <small class="fw-bold text-muted uppercase">TOP CIDADES (Top 5)</small>
                            <Info :size="14" class="text-muted" />
                        </div>
                        <div style="height: 180px;">
                            <Bar :data="chartCidadesTop5" 
                                :options="{ 
                                    indexAxis: 'y', 
                                    responsive: true, 
                                    maintainAspectRatio: false, 
                                    plugins: { legend: { display: false } },
                                    scales: { y: { ticks: { font: { size: 9 } } } }
                                }" />
                        </div>
                    </div>
                </div>
            </div>

            <!-- RODAPÉ -->
            <div class="row g-2">
                <div class="col-md-5">
                    <div class="card border-0 shadow-sm p-3 rounded-4 bg-white h-100">
                        <small class="fw-bold text-muted uppercase mb-3 d-block">PARETO DE VISCOSIDADE</small>
                        <div style="height: 220px;"><Bar :data="chartViscosidade" :options="{ responsive: true, maintainAspectRatio: false, plugins: { legend: { display: false } }, scales: { y: { beginAtZero: true }, y1: { position:'right', min:0, max:110, grid: { display: false } } } }" /></div>
                    </div>
                </div>
                <div class="col-md-7">
                    <div class="row g-2 h-100">
                        <div v-for="g in [{l:'BÁSICO', k:'basico'}, {l:'SEGMENTO', k:'segmento'}, {l:'NORMAS', k:'norma'}]" :key="g.k" class="col-4">
                            <div class="card border-0 shadow-sm p-3 rounded-4 bg-white h-100 text-center">
                                <small class="fw-bold text-muted d-block text-start mb-2">{{ g.l }}</small>
                                <div style="height: 160px;"><Doughnut :data="{ labels: detalhada.graficos[g.k].map(i => i.label), datasets: [{ data: detalhada.graficos[g.k].map(i => i.litros), backgroundColor: orangePalette }] }" :options="doughnutOptionsStandard" /></div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <!-- MODAL DINÂMICO EXCLUSIVO -->
    <div v-if="showModal" class="modal-overlay d-flex align-items-center justify-content-center" @click.self="showModal = false">
        <!-- Largura de 950px se for Estados ou Cidades -->
        <div class="modal-content bg-white p-5 rounded-4 shadow-lg position-relative" 
            :style="{ width: (modalChartType === 'bar' || modalChartType === 'bar-cities') ? '950px' : '550px' }">
            
            <button @click="showModal = false" class="btn-close-custom"><X :size="20"/></button>
            <div class="text-center mb-4"><h4 class="fw-bold text-dark m-0">{{ modalTitle }}</h4></div>
            
            <div v-if="isModalLoading" class="text-center p-5"><div class="spinner-border text-orange"></div></div>
            
            <div v-else style="height: 400px;">
                <!-- GRÁFICO EXCLUSIVO CIDADES: Vertical e Largo -->
                <Bar v-if="modalChartType === 'bar-cities'" 
                    :data="chartCidadesFull" 
                    :options="{ 
                        responsive: true, 
                        maintainAspectRatio: false, 
                        plugins: { legend: { display: false } },
                        scales: { x: { ticks: { font: { size: 8 }, maxRotation: 45, minRotation: 45 } } } 
                    }" />

                <!-- GRÁFICO EXCLUSIVO ESTADOS: Vertical e Largo -->
                <Bar v-else-if="modalChartType === 'bar'" 
                    :data="chartEstadosFull" 
                    :options="{ responsive: true, maintainAspectRatio: false, plugins: { legend: { display: false } }, scales: { x: { ticks: { font: { size: 8 }, maxRotation: 45, minRotation: 45 } } } }" />
                
                <!-- MARCAS / MODELOS: Pizza Padrão -->
                <Pie v-else-if="modalData" 
                    :data="{ labels: modalData.map(i => i.label), datasets: [{ data: modalData.map(i => i.litros), backgroundColor: orangePalette }] }" 
                    :options="{ responsive: true, maintainAspectRatio: false }" />
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
.icon-header { width: 48px; height: 48px; background-color: #e97332; border-radius: 12px; display: flex; align-items: center; justify-content: center; }
.btn-detail { background-color: #eee; color: #888; border: none; transition: 0.3s; }
.btn-detail:hover { background-color: #e97332; color: white; }
.filter-label { font-size: 10px; font-weight: 800; color: #333; margin-bottom: 2px; display: block; }
.card-white-kpi { cursor: pointer; transition: 0.2s; border: 1px solid transparent; }
.card-white-kpi:hover { border-color: #e97332; background: #fffdfc !important; }
.loader-overlay { position: absolute; top:0; left:0; width:100%; height:100%; background: rgba(255,255,255,0.7); display:flex; align-items:center; justify-content:center; z-index: 10; }
.icon-circle-print { width: 45px; height: 45px; border-radius: 50%; border: 2.5px solid #e97332; display: flex; align-items: center; justify-content: center; }
.modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.6); z-index: 9999; backdrop-filter: blur(4px); }
.btn-red-clear { background-color: #ff0000; font-size: 11px; border: none; border-radius: 8px; height: 32px; }
.last-no-border:last-child { border-right: none !important; }
.btn-close-custom { position: absolute; top: 20px; right: 20px; border: none; background: none; color: #888; }
input[type=number]::-webkit-inner-spin-button { -webkit-appearance: none; }
</style>
