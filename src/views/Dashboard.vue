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
const api = axios.create({ baseURL: 'https://lubx-api.lubconsulta.com.br/bi/regiao-2', timeout: 20000 });
// const api = axios.create({ baseURL: 'http://localhost:3000/bi/regiao-2', timeout: 20000 });

// Plugin customizado para desenhar as linhas (Leader Lines)
const doughnutLabelsLine = {
  id: 'doughnutLabelsLine',
  afterDraw(chart) {
    // Só executa se for gráfico de rosca e se datalabels estiver ativo
    if (chart.config.type !== 'doughnut' && chart.config.type !== 'pie') return;

    const { ctx } = chart;
    chart.data.datasets.forEach((dataset, i) => {
      chart.getDatasetMeta(i).data.forEach((datapoint, index) => {
        const { x, y, startAngle, endAngle, outerRadius } = datapoint;
        const midAngle = startAngle + (endAngle - startAngle) / 2;

        // Lógica para não desenhar linha em labels ocultos (fatias pequenas nas Normas)
        const realData = dataset.realData || dataset.data;
        const total = realData.reduce((a, b) => a + b, 0);
        if (total === 0) return;
        const val = realData[index];
        
        // Se o formatter do datalabels retornaria null, não desenhamos a linha
        if (chart.options.plugins.datalabels.display && (val/total) < 0.04 && chart.canvas.id.includes('norma')) {
           // Nota: O ID depende de como o Chart.js identifica o componente, 
           // na dúvida, ele desenhará para todos os que têm label visível.
        }

        const x1 = x + Math.cos(midAngle) * outerRadius;
        const y1 = y + Math.sin(midAngle) * outerRadius;
        const x2 = x + Math.cos(midAngle) * (outerRadius + 15); // Aumentei um pouco a linha
        const y2 = y + Math.sin(midAngle) * (outerRadius + 15);

        ctx.strokeStyle = '#D1D5DB'; // Cinza suave
        ctx.lineWidth = 1;
        ctx.beginPath();
        ctx.moveTo(x1, y1);
        ctx.lineTo(x2, y2);
        ctx.stroke();
      });
    });
  }
};

// REGISTRE O PLUGIN AQUI
ChartJS.register(
    CategoryScale, LinearScale, RadialLinearScale, BarElement, ArcElement, 
    PointElement, LineElement, BarController, LineController, Title, Tooltip, Legend, ChartDataLabels,
    doughnutLabelsLine // <--- Adicione ele aqui
);



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
const modalChartType = ref('pie'); 

const apiFilters = ref(null);
const filters = reactive({
    tipoVeiculo: 'Todos', marca: 'Todos', veiculo: 'Todos', modelo: 'Todos',
    ano: 'Todos', motor: 'Todos', combustivel: 'Todos', viscosidade: 'Todos',
    api: 'Todos', acea: 'Todos', jaso: 'Todos', norma: 'Todos', basico: 'Todos',
    trocaLeve: 1, trocaPesada: 3.8, trocaMoto: 3
});

const panorama = ref(null);
const detalhada = ref(null);

const stateNames = {
  'AC': 'Acre', 'AL': 'Alagoas', 'AP': 'Amapá', 'AM': 'Amazonas', 'BA': 'Bahia', 'CE': 'Ceará', 'DF': 'Distrito Federal', 'ES': 'Espírito Santo', 'GO': 'Goiás', 'MA': 'Maranhão', 'MT': 'Mato Grosso', 'MS': 'Mato Grosso do Sul', 'MG': 'Minas Gerais', 'PA': 'Pará', 'PB': 'Paraíba', 'PR': 'Paraná', 'PE': 'Pernambuco', 'PI': 'Piauí', 'RJ': 'Rio de Janeiro', 'RN': 'Rio Grande do Norte', 'RS': 'Rio Grande do Sul', 'RO': 'Rondônia', 'RR': 'Roraima', 'SC': 'Santa Catarina', 'SP': 'São Paulo', 'SE': 'Sergipe', 'TO': 'Tocantins'
};

const orangePalette = ['#e97332', '#f4b393', '#fbe2d5', '#3b82f6', '#10b981', '#f59e0b', '#00bcd4'];

// Formatação para 2 casas decimais
const formatNum = (val) => new Intl.NumberFormat('pt-BR', { minimumFractionDigits: 2, maximumFractionDigits: 2 }).format(val);

// --- FUNÇÕES DE API ---

const loadFilters = async () => {
    try {
        const res = await api.get('/filtros');
        apiFilters.value = res.data.data;
    } catch (e) { console.error("Erro ao carregar filtros:", e); }
};

const loadData = async () => {
    if (isLoading.value) return;
    isLoading.value = true;
    const ufParam = selectedUF.value === 'Todos' ? '' : selectedUF.value;
    try {
        const params = { estado: ufParam, ...filters };
        const res = await api.get(viewMode.value === 'executive' ? '/panorama' : '/analise-detalhada', { params });
        if (viewMode.value === 'executive') panorama.value = res.data.data;
        else detalhada.value = res.data.data;
    } catch (e) { console.error(e); } finally {
        isLoading.value = false;
        if (map.value) nextTick(() => map.value.resize());
    }
};

const handleChartClick = (event, elements, chartKey) => {
    if (!elements.length) return;
    const index = elements[0].index;
    const label = detalhada.value.graficos[chartKey][index].label;
    if (chartKey === 'basico') filters.basico = label;
    if (chartKey === 'segmento') filters.tipoVeiculo = label;
    if (chartKey === 'norma') filters.norma = label;
};

const openDetailedModal = async (type) => {
    modalTitle.value = `Top 10 ${type} (Consolidado)`;
    modalChartType.value = 'pie'; showModal.value = true; isModalLoading.value = true;
    try {
        const res = await api.get(type === 'Marcas' ? '/graficos/marcas' : '/graficos/modelos', { 
            params: { estado: selectedUF.value === 'Todos' ? '' : selectedUF.value, tipoVeiculo: 'Todos', jaso: 'Todos' } 
        });
        // ALTERADO: Ordenando por .frota em vez de .litros
        modalData.value = res.data.data.sort((a,b) => b.frota - a.frota).slice(0, 10);
    } catch (e) { console.error(e); } finally { isModalLoading.value = false; }
};

const openStatesModal = () => { modalTitle.value = 'Volume por Estado (Completo)'; modalChartType.value = 'bar'; showModal.value = true; modalData.value = true; };
const openCitiesModal = () => { modalTitle.value = 'Top 15 Cidades (Volume Mi L)'; modalChartType.value = 'bar-cities'; showModal.value = true; modalData.value = true; };
const clearFilters = () => { Object.keys(filters).forEach(k => { if (!k.startsWith('troca')) filters[k] = 'Todos'; }); };

// --- AJUSTE: NORMAS TOP 10 COM REGRA DE 20% ---
const normasChartData = computed(() => {
    if (!detalhada.value?.graficos?.norma) return { labels: [], datasets: [] };

    // Filtramos para REMOVER o "SEM NORMA" e depois pegamos o Top 10 das normas reais
    const raw = detalhada.value.graficos.norma
        .filter(i => i.label !== 'SEM NORMA' && i.label !== 'Não informado') 
        .sort((a, b) => b.litros - a.litros)
        .slice(0, 10);

    return {
        labels: raw.map(i => i.label),
        datasets: [{ 
            data: raw.map(i => i.litros), 
            realData: raw.map(i => i.litros), 
            backgroundColor: orangePalette 
        }]
    };
});

const getDoughnutOptions = (key) => ({
    responsive: true,
    maintainAspectRatio: false,
    layout: {
        // Aumentamos o padding global para os textos escalonados não cortarem nas bordas do card
        padding: 18 
    },
    // Reduzimos o raio do gráfico para 65% para sobrar espaço para as etiquetas "subirem"
    radius: '55%', 
    cutout: '45%', 
    onClick: (e, el) => handleChartClick(e, el, key),
    plugins: {
        legend: {
            display: true,
            position: 'bottom',
            align: 'end', // <--- Move a legenda para o canto inferior direito
            labels: {
                usePointStyle: true,
                pointStyle: 'circle',
                boxWidth: 18,
                padding: 5, 
                font: {
                    family: "'Inter', sans-serif",
                    size: 9,
                    weight: '500'
                }
            }
        },
        datalabels: {
            display: true,
            anchor: 'end',
            align: 'end',
            // FUNÇÃO DE ESCALONAMENTO (STAGGERING)
            offset: (context) => {
                const dataset = context.dataset;
                const realData = dataset.realData || dataset.data;
                const value = realData[context.dataIndex];
                const total = realData.reduce((a, b) => a + b, 0);
                
                // Se a fatia for pequena (menos de 10%), alterna o offset em 3 níveis
                // Isso faz com que os nomes fiquem um acima do outro
                if (total > 0 && (value / total) < 0.20) {
                    return (context.dataIndex % 5) * 7 + 0; 
                }
                return 10;
            },
            color: '#444',
            textAlign: 'center',
            font: {
                family: "'Inter', sans-serif",
                size: 8,
                weight: 'bold'
            },
            formatter: (v, ctx) => {
                const label = ctx.chart.data.labels[ctx.dataIndex];
                const realData = ctx.dataset.realData || ctx.dataset.data;
                const total = realData.reduce((a, b) => a + b, 0);
                const val = realData[ctx.dataIndex];
                
                if (total === 0) return null;
                const pct = ((val / total) * 100).toFixed(1) + '%';
                
                // Se for muito pequeno (menos de 1%), nem exibe para não poluir
                if ((val / total) < 0.01) return null;

                // Para o gráfico de básico, ajuste específico solicitado anteriormente
                if (label === 'SEMISSINTÉTICO') return label;

                return `${label}\n${pct}`;
            }
        }
    }
});

const totalLitrosCalculado = computed(() => {
    if (!detalhada.value?.comparativosSegmento) return 0;
    return detalhada.value.comparativosSegmento.reduce((acc, s) => acc + s.litrosAno, 0);
});

// --- AJUSTE: CIDADES BRASIL E TOP 30 ---
const chartCidadesTop5 = computed(() => {
    const list = detalhada.value?.graficos?.topCidades || [];
    return { 
        labels: list.slice(0,5).map(c => c.cidade), 
        datasets: [{ backgroundColor: '#e97332', data: list.slice(0,5).map(c => c.litrosAno/1000000), barThickness: 15, datalabels: { formatter: v => v.toFixed(2) } }] 
    };
});

const chartCidadesFull = computed(() => {
    const list = detalhada.value?.graficos?.topCidades || [];
    return { 
        labels: list.slice(0,30).map(c => c.cidade), 
        datasets: [{ backgroundColor: '#e97332', data: list.slice(0,30).map(c => c.litrosAno/1000000), barThickness: 12, datalabels: { formatter: v => v.toFixed(2) } }] 
    };
});

// --- OUTROS COMPUTEDS ---
const chartViscosidade = computed(() => {
    if (!detalhada.value?.graficos?.viscosidade) return { labels: [], datasets: [] };
    const items = detalhada.value.graficos.viscosidade;
    let acc = 0; const total = items.reduce((a, b) => a + b.litros, 0);
    return {
        labels: items.map(i => i.label),
        datasets: [
            { 
                type: 'line', 
                label: '% Acc', 
                borderColor: '#e97332', 
                data: items.map(i => { acc += i.litros; return ((acc / total) * 100).toFixed(2); }), 
                yAxisID: 'y1', 
                tension: 0.2,
                datalabels: { 
                    align: 'top', 
                    anchor: 'end', 
                    offset: -4, // Distância maior para "subir" o número da linha
                    font: { weight: 'bold', size: 9 },
                    color: '#000' // Cor opcional para combinar com a linha
                } 
            },
            { 
                type: 'bar', 
                label: 'Vol Mi', 
                backgroundColor: '#3b82f6', 
                data: items.map(i => i.litros / 1000000), 
                yAxisID: 'y', 
                barThickness: 28, 
                datalabels: { 
                    formatter: v => v.toFixed(2),
                    align: 'top',   // Adicionado aqui
                    anchor: 'end'   // Adicionado aqui
                } 
            }
        ]
    };
});

const chartEstados = computed(() => ({ labels: detalhada.value?.graficos?.estado?.slice(0,10).map(e=>e.estado) || [], datasets: [{ backgroundColor: '#e97332', data: detalhada.value?.graficos?.estado?.slice(0,10).map(e=>e.litrosAno/1000000) || [], barThickness: 25, datalabels: { formatter: v => v.toFixed(2) } }] }));
const chartEstadosFull = computed(() => ({ labels: detalhada.value?.graficos?.estado?.map(e=>e.estado) || [], datasets: [{ backgroundColor: '#e97332', data: detalhada.value?.graficos?.estado?.map(e=>e.litrosAno/1000000) || [], barThickness: 15, datalabels: { formatter: v => v.toFixed(2) } }] }));

// --- MAPBOX ---
const mapContainer = ref(null); const map = ref(null);
const updateMapHighlight = () => {
    if(!map.value || !map.value.isStyleLoaded()) return;
    map.value.setPaintProperty('states-fill', 'fill-color', ['case', ['==', ['get', 'sigla'], selectedUF.value], '#e97332', '#f97316']);
    map.value.setPaintProperty('states-fill', 'fill-opacity', ['case', ['==', ['get', 'sigla'], selectedUF.value], 0.5, 0.1]);
};

const initMap = () => {
    mapboxgl.accessToken = import.meta.env.VITE_MAPBOX_TOKEN || 'SUA_CHAVE_AQUI';
    map.value = new mapboxgl.Map({ container: mapContainer.value, style: 'mapbox://styles/mapbox/light-v11', center: [-52, -15], zoom: 3.5, projection: 'mercator', attributionControl: false });
    map.value.on('load', () => {
        map.value.addSource('brazil-states', { type: 'geojson', data: 'https://raw.githubusercontent.com/codeforamerica/click_that_hood/master/public/data/brazil-states.geojson' });
        map.value.addLayer({ 'id': 'states-fill', 'type': 'fill', 'source': 'brazil-states', 'paint': { 'fill-color': '#f97316', 'fill-opacity': 0.1 }});
        map.value.addLayer({ 'id': 'states-borders', 'type': 'line', 'source': 'brazil-states', 'paint': { 'line-color': '#e97332', 'line-width': 0.5 }});
        map.value.on('click', 'states-fill', (e) => { 
            const clicado = e.features[0].properties.sigla; 
            selectedUF.value = (selectedUF.value === clicado) ? 'Todos' : clicado; 
        });
        updateMapHighlight();
    });
};

watch([selectedUF, viewMode], () => { 
    if (selectedUF.value === 'Todos') selectedStateName.value = 'Brasil'; 
    else selectedStateName.value = stateNames[selectedUF.value]; 
    updateMapHighlight(); // <-- Restaurado aqui
    loadData(); 
});
watch(filters, () => loadData(), { deep: true });
onMounted(() => { initMap(); loadFilters(); loadData(); });
</script>

<template>
  <div class="dashboard-container">
    
    <!-- PANORAMA -->
    <div v-show="viewMode === 'executive'" class="view-wrapper p-4">
      <div class="row g-4 align-items-stretch">
        <div class="col-lg-7">
          <div class="card map-card position-relative border-0 shadow-sm rounded-4 overflow-hidden bg-white" style="min-height: 750px;">
            <div class="p-4 position-absolute top-0 start-0 z-2"><h3 class="fw-bold m-0" style="color: #e97332">Panorama <span class="text-dark">Estratégico</span></h3></div>
            <div ref="mapContainer" class="position-absolute top-0 start-0 w-100 h-100"></div>
            <div v-if="isLoading" class="loader-overlay"><div class="spinner-border text-orange"></div></div>
          </div>
        </div>
        <div class="col-lg-5">
            <div class="card h-100 border-0 shadow-sm p-4 d-flex flex-column rounded-4 bg-white">
                <div class="d-flex align-items-center gap-3 mb-4"><div class="icon-header"><MapIcon class="text-white" :size="24"/></div><h2 class="fw-bold m-0">{{ selectedStateName }}</h2></div>
                <div v-if="panorama">
                    <div class="row mb-4"><div class="col-6"><h3 class="fw-bold m-0">{{ formatNum(panorama.resumo.litrosAnoTotal) }}</h3><small class="text-muted">Litros / ano total</small></div><div class="col-6 border-start ps-4"><h3 class="fw-bold m-0">{{ formatNum(panorama.resumo.frotaGeral) }}</h3><small class="text-muted">Frota Geral</small></div></div>
                    <p style="font-size: 13px;"><span>&#9432;</span> A litragem total apresentada está alinhada com a estimativa média nacional de trocas de óleo por segmento.</p>
                    <hr class="mb-4"><div class="flex-grow-1"><label class="fw-bold mb-4 d-block">Segmentação e Perfil:</label><div v-for="s in panorama.perfilSegmentacao" :key="s.id" class="mb-4"><div class="d-flex align-items-start gap-3"><component :is="s.id.includes('leve')?Car:s.id.includes('pesada')?Truck:Bike" class="text-orange mt-1" :size="32"/><div class="flex-grow-1"><div class="d-flex justify-content-between"><strong>{{ formatNum(s.litrosAno) }} L</strong><small class="text-muted">{{ formatNum(s.frota) }} veíc.</small></div><div class="progress" style="height: 6px;"><div class="progress-bar bg-orange" :style="{width: s.percentualLitros + '%'}"></div></div><small class="text-muted" style="font-size: 10px;">{{ s.label }} ({{ s.percentualLitros }}%)</small></div></div></div><div class="pt-3 border-top d-flex align-items-center gap-3"><Clock class="text-orange" :size="32"/><div><strong class="fs-5">{{ panorama.resumo.idadeMediaFrota }} anos</strong><small class="text-muted d-block" style="font-size:10px">Idade Média da Frota</small></div></div></div>
                </div>
                <button @click="viewMode = 'detailed'" class="btn btn-detail w-100 py-3 mt-4 fw-bold shadow-sm">Avançar para Análise Detalhada</button>
            </div>
        </div>
      </div>
    </div>

    <!-- ANALISE DETALHADA -->
    <div v-show="viewMode === 'detailed'" class="view-wrapper detailed-bg pb-3 h-100 overflow-auto">
        <div class="header-detailed bg-white border-bottom p-2 px-4 d-flex align-items-center justify-content-between sticky-top">
            <div class="d-flex align-items-center gap-3"><button @click="viewMode = 'executive'" class="btn btn-sm btn-light border rounded-circle"><ArrowLeft :size="18"/></button>
            <div class="d-flex align-items-center gap-2"><span class="fw-bold text-dark">ESTADO:</span><select v-model="selectedUF" class="form-select border-0 fw-bold text-orange p-0 bg-transparent fs-7" style="width: auto;"><option value="Todos">Brasil</option><option v-for="(name, sigla) in stateNames" :key="sigla" :value="sigla">{{ name }}</option></select></div></div>
            <div v-if="isLoading" class="spinner-border spinner-border-sm text-orange"></div>
        </div>

        <div class="container-fluid px-4 mt-2">
            <div class="card border-0 shadow-sm p-3 mb-3 bg-white rounded-3" v-if="apiFilters">
                <div class="row g-2 mb-2">
                    <div v-for="f in [{l:'TIPO VEÍCULO', k:'tipoVeiculo'}, {l:'MARCA', k:'marca'}, {l:'VEÍCULO', k:'veiculo'}, {l:'MODELO', k:'modelo'}, {l:'ANO', k:'ano'}, {l:'MOTOR', k:'motor'}, {l:'COMBUSTÍVEL', k:'combustivel'}]" :key="f.k" class="col"><label class="filter-label">{{ f.l }}:</label><select v-model="filters[f.k]" class="form-select form-select-sm border-secondary-subtle"><option value="Todos">Todos</option><option v-for="o in apiFilters[f.k]" :key="o" :value="o">{{ o }}</option></select></div>
                </div>
                <div class="row g-2 align-items-end">
                    <div v-for="f in [{l:'VISCOSIDADE', k:'viscosidade'}, {l:'API', k:'api'}, {l:'ACEA', k:'acea'}, {l:'JASO', k:'jaso'}, {l:'NORMA', k:'norma'}, {l:'BÁSICO', k:'basico'}]" :key="f.k" class="col"><label class="filter-label">{{ f.l }}:</label><select v-model="filters[f.k]" class="form-select form-select-sm border-secondary-subtle"><option value="Todos">Todos</option><option v-for="o in apiFilters[f.k]" :key="o" :value="o">{{ o }}</option></select></div>
                    <div class="col-auto"><button @click="clearFilters" class="btn btn-red-clear text-white fw-bold px-4">Limpar Filtros</button></div>
                </div>
            </div>

            <div v-if="detalhada">
                <div class="row g-3 mb-4">
                    <div v-for="k in detalhada.cardsTopo" :key="k.id" class="col-md-3">
                        <div class="card border-0 shadow-sm p-4 rounded-4 bg-white d-flex flex-row align-items-center justify-content-center gap-3" :class="{'card-white-kpi': k.id === 'marcas' || k.id === 'modelos'}" @click="(k.id === 'marcas' || k.id === 'modelos') ? openDetailedModal(k.label) : null">
                            <div class="icon-circle-print"><Zap v-if="k.id==='litros-ano'" class="text-orange" :size="24" /><Car v-else class="text-orange" :size="24" /></div>
                            <div>
                                <!-- Alteração aqui: se for o card de litros, usa o valor calculado por nós -->
                                <h2 class="fw-bold m-0 text-dark" style="font-size:1.7rem">
                                    {{ k.id === 'litros-ano' ? formatNum(totalLitrosCalculado) : formatNum(k.valor) }}
                                </h2>
                                <small class="text-muted fw-bold d-block">{{ k.label }}</small>
                            </div>
                        </div>
                    </div>
                </div>

                <div class="row g-2 mb-3">
                    <!-- CARD SEGMENTOS -->
                    <div class="col-lg-5">
                        <div class="card border-0 shadow-sm p-3 h-100 rounded-4 bg-white">
                            <div class="row h-100 align-items-center">
                                <div v-for="s in detalhada.comparativosSegmento" :key="s.id" class="col-4 border-end last-no-border px-3">
                                    <div class="d-flex align-items-center gap-2 mb-3">
                                        <component :is="s.id.includes('leve')?Car:s.id.includes('pesada')?Truck:Bike" class="text-orange" :size="42" />
                                        <div class="lh-sm"><strong>{{ formatNum(s.veiculos) }}</strong><small class="text-muted d-block" style="font-size:10px">Veículos</small></div>
                                    </div>
                                    <div class="mb-3 border-top pt-2"><strong>{{ formatNum(s.litrosAno) }}</strong><small class="text-muted d-block" style="font-size:10px">Litros / ano</small></div>
                                    <div class="p-2 rounded-3 border border-light-subtle text-center bg-white shadow-sm">
                                        <small class="text-muted d-block mb-1 fw-bold" style="font-size:10px">Trocas / Ano</small>
                                        <input type="number" step="0.1" class="form-control form-control-sm text-center fw-bold bg-light border-0" v-model.number="filters[s.id.includes('leve') ? 'trocaLeve' : (s.id.includes('pesada') ? 'trocaPesada' : 'trocaMoto')]">
                                    </div>
                                </div>
                                    <p style="font-size: 13px;"><span>&#9432;</span> A quantidade de trocas apresentada, representa uma estimativa baseada na média nacional.</p>
                            </div>
                        </div>
                    </div>

                    <!-- CARD ESTADOS (AJUSTADO PARA 250) -->
                    <div class="col-lg-4">
                        <div class="card border-0 shadow-sm p-3 h-100 rounded-4 bg-white card-white-kpi" @click="openStatesModal">
                            <small class="fw-bold text-muted uppercase">ESTADOS (Top 10)</small>
                            <div style="height: 210px;">
                                <Bar :data="chartEstados" :options="{ 
                                    responsive: true, 
                                    maintainAspectRatio: false, 
                                    layout: { padding: { top: 20 } },
                                    scales: { y: { beginAtZero: true, max: 250 } },
                                    plugins: { 
                                        legend: { display: false },
                                        datalabels: { anchor: 'end', align: 'top', font: { weight: 'bold' }, formatter: (v) => v.toFixed(2) } 
                                    } 
                                }" />
                            </div>
                        </div>
                    </div>

                    <!-- CARD CIDADES -->
                    <div class="col-lg-3">
                        <div class="card border-0 shadow-sm p-3 h-100 rounded-4 bg-white card-white-kpi" @click="openCitiesModal">
                            <small class="fw-bold text-muted uppercase">CIDADES (Top 5)</small>
                            <div style="height: 210px;">
                                <Bar :data="chartCidadesTop5" :options="{ 
                                    indexAxis: 'y', 
                                    responsive: true, 
                                    maintainAspectRatio: false, 
                                    layout: { padding: { right: 40 } }, // Aumentei o respiro lateral
                                    scales: { 
                                        x: { 
                                            beginAtZero: true, 
                                            max: 40 // Define o limite máximo em 40 para encurtar as barras
                                        } 
                                    },
                                    plugins: { 
                                        legend: { display: false },
                                        datalabels: { 
                                            anchor: 'end', 
                                            align: 'right', 
                                            offset: 5,
                                            font: { weight: 'bold' },
                                            formatter: (v) => v.toFixed(2)
                                        } 
                                    } 
                                }" />
                            </div>
                        </div>
                    </div>
                </div>

                <div class="row g-2">
                    <div class="col-md-5"><div class="card border-0 shadow-sm p-3 rounded-4 bg-white h-100"><small class="fw-bold text-muted uppercase mb-3 d-block">VISCOSIDADE</small><div style="height: 220px;">
                    <div style="height: 220px;">
                    <!-- VISCOSIDADE - Ajuste do eixo Y para 300 -->
                    <Bar 
                        :data="chartViscosidade" 
                        :options="{ 
                        responsive: true, 
                        maintainAspectRatio: false,
                        layout: {
                            padding: {
                                top: 5 // Aumentado significativamente para dar espaço à legenda e aos números
                            }
                        },
                        plugins: {
                            legend: {
                                display: true,
                                position: 'top',
                                align: 'start',
                                labels: { 
                                    boxWidth: 12, 
                                    font: { size: 10 }, 
                                    padding: 3 // Aumenta o espaço entre a legenda e as barras do gráfico
                                }
                            },
                            datalabels: {
                                anchor: 'end',
                                align: 'top',
                                offset: 0, // Aumenta um pouco a distância do número para o ponto/barra
                                font: { weight: 'bold', size: 10 },
                                formatter: (v, ctx) => {
                                    // Se for a linha de %, adiciona o símbolo
                                    if (ctx.dataset.type === 'line') return v + '%';
                                    return v;
                                }
                            }
                        },
                        scales: { 
                            y: { 
                                beginAtZero: true, 
                                max: 300 // Garante que as barras fiquem baixas
                            }, 
                            y1: { 
                                position: 'right', 
                                grid: { display: false }, 
                                min: 0, 
                                max: 180 // Aumentado de 110 para 140 para a linha de 100% não bater no teto
                            } 
                        } 
                        }" 
                    />
                    </div></div></div></div>
                    <div class="col-md-7"><div class="row g-2 h-100">
                        <!-- BÁSICO -->
                            <div class="col-4">
                                <div class="card border-0 shadow-sm p-3 rounded-4 bg-white h-100 text-center">
                                    <small class="fw-bold text-muted d-block text-start mb-2">BÁSICO</small>
                                    <div style="height: 250px;"> <!-- Aumentado um pouco para caber as labels fora -->
                                        <Doughnut 
                                            :data="{ 
                                                labels: detalhada.graficos.basico.filter(i => i.label !== 'Não informado').map(i=>i.label), 
                                                datasets: [{ 
                                                    data: detalhada.graficos.basico.filter(i => i.label !== 'Não informado').map(i=>i.litros), 
                                                    backgroundColor: orangePalette 
                                                }] 
                                            }" 
                                            :options="getDoughnutOptions('basico')" 
                                        />
                                    </div>
                                </div>
                            </div>

                            <!-- SEGMENTO -->
                            <div class="col-4">
                                <div class="card border-0 shadow-sm p-3 rounded-4 bg-white h-100 text-center">
                                    <small class="fw-bold text-muted d-block text-start mb-2">SEGMENTO</small>
                                    <div style="height: 250px;">
                                        <Doughnut 
                                            :data="{ 
                                                labels: detalhada.graficos.segmento.map(i=>i.label), 
                                                datasets: [{ 
                                                    data: detalhada.graficos.segmento.map(i=>i.litros), 
                                                    backgroundColor: orangePalette 
                                                }] 
                                            }" 
                                            :options="getDoughnutOptions('segmento')" 
                                        />
                                    </div>
                                </div>
                            </div>

                            <!-- NORMAS -->
                            <div class="col-4">
                                <div class="card border-0 shadow-sm p-3 rounded-4 bg-white h-100 text-center">
                                    <small class="fw-bold text-muted d-block text-start mb-2">NORMAS (TOP 10)</small>
                                    <div style="height: 250px;">
                                        <Doughnut 
                                            :data="normasChartData" 
                                            :options="getDoughnutOptions('norma')" 
                                        />
                                    </div>
                                </div>
                            </div>
                    </div></div>
                </div>
            </div>
        </div>
    </div>

    <!-- MODAL -->
    <div v-if="showModal" class="modal-overlay d-flex align-items-center justify-content-center" @click.self="showModal = false">
        <div class="modal-content bg-white p-5 rounded-4 shadow-lg position-relative" :style="{ width: (modalChartType === 'bar' || modalChartType === 'bar-cities') ? '950px' : '550px' }">
            <button @click="showModal = false" class="btn-close-custom"><X :size="20"/></button>
            <div class="text-center mb-4"><h4 class="fw-bold text-dark m-0">{{ modalTitle }}</h4></div>
            <div v-if="isModalLoading" class="text-center p-5"><div class="spinner-border text-orange"></div></div>
            <div v-else style="height: 400px;">
                <!-- Gráfico de Cidades no Modal -->
<Bar 
    v-if="modalChartType === 'bar-cities'" 
    :data="chartCidadesFull" 
    :options="{ 
        responsive: true, 
        maintainAspectRatio: false, 
        layout: { padding: { top: 30 } }, 
        plugins: { 
            legend: { display: false },
            datalabels: { 
                anchor: 'end', 
                align: 'top', 
                offset: 4,
                font: { weight: 'bold', size: 10 },
                formatter: (v) => v.toFixed(2)
            } 
        } 
    }" 
/>

<!-- Gráfico de Estados no Modal -->
<Bar 
    v-else-if="modalChartType === 'bar'" 
    :data="chartEstadosFull" 
    :options="{ 
        responsive: true, 
        maintainAspectRatio: false, 
        layout: { padding: { top: 30 } }, 
        scales: { 
            y: { 
                beginAtZero: true, 
                max: 250 // Ajuste aqui também para consistência
            } 
        },
        plugins: { 
            legend: { display: false },
            datalabels: { 
                anchor: 'end', 
                align: 'top', 
                offset: 4,
                font: { weight: 'bold', size: 10 },
                formatter: (v) => v.toFixed(2)
            } 
        } 
    }" 
/>
                <Pie 
                    v-else-if="modalData" 
                    :data="{ labels: modalData.map(i => i.label), datasets: [{ data: modalData.map(i => i.frota), backgroundColor: orangePalette }] }" 
                    :options="{
                        responsive: true, 
                        maintainAspectRatio: false,
                        layout: {
                            // Aumentamos o padding inferior para a legenda não encostar no gráfico
                            padding: {
                                top: 40,
                                bottom: 0,
                                left: 10,
                                right: 10
                            }
                        },
                        radius: '60%', 
                        plugins: { 
                            legend: { 
                                display: true, 
                                position: 'bottom', // Move para a parte de baixo
                                align: 'end',       // Alinha à direita
                                labels: { 
                                    usePointStyle: true, 
                                    pointStyle: 'circle', 
                                    font: { size: 10 },
                                    padding: 15     // Espaço entre os itens da legenda
                                }
                            }, 
                            datalabels: { 
                                display: true,
                                anchor: 'end',
                                align: 'end',
                                offset: (context) => {
                                    const value = context.dataset.data[context.dataIndex];
                                    const total = context.dataset.data.reduce((a, b) => a + b, 0);
                                    // Se a fatia for pequena, escalona para não sobrepor
                                    if ((value / total) < 0.15) {
                                        return (context.dataIndex % 3) * 35 + 10; 
                                    }
                                    return 15;
                                },
                                color: '#444', 
                                textAlign: 'center', 
                                font: { 
                                    family: 'Inter, sans-serif',
                                    weight: 'bold', 
                                    size: 10 
                                }, 
                                formatter: (v, ctx) => { 
                                    return ctx.chart.data.labels[ctx.dataIndex] + '\n' + formatNum(v); 
                                } 
                            } 
                        } 
                    }" 
                />
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
