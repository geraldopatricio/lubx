<script setup>
import { ref, computed, onMounted, watch, reactive } from 'vue';
import axios from 'axios';
import { 
  Car, Bike, Truck, Calculator, Info, Filter, 
  MapPin, Eraser, List, ExternalLink, ArrowLeft 
} from 'lucide-vue-next'; 
import mapboxgl from 'mapbox-gl';
import 'mapbox-gl/dist/mapbox-gl.css';

// --- CONFIGURAÇÃO API ---
// const api = axios.create({ baseURL: 'http://localhost:3000/bi/oportunidades', timeout: 60000 });
const api = axios.create({ baseURL: 'https://lubx-api.lubconsulta.com.br/bi/oportunidades', timeout: 60000 });

// --- ESTADOS REATIVOS ---
const isLoading = ref(true);
const selectedUF = ref('Todos');
const apiFilters = ref(null);

const filters = reactive({
    viscosidade: 'Todos',
    api: 'Todos',
    acea: 'Todos',
    jaso: 'Todos',
    basico: 'Todos',
    tipoVeiculo: ['LEVE']
});

const simuladorShare = reactive({
  suaVendaAtualLitros: 0,
  mesesCorridos: 0,
  shareDesejado: 0
});

const overviewData = ref({ potencialConsumoLitrosAno: 0, frotaEfetiva: 0 });
const simuladorTrocasData = ref([]);
const tableData = ref([]);

const stateNames = {
  'AC': 'Acre', 'AL': 'Alagoas', 'AM': 'Amazonas', 'AP': 'Amapá', 'BA': 'Bahia', 'CE': 'Ceará', 'DF': 'Distrito Federal', 'ES': 'Espírito Santo', 'GO': 'Goiás', 'MA': 'Maranhão', 'MT': 'Mato Grosso', 'MS': 'Mato Grosso do Sul', 'MG': 'Minas Gerais', 'PA': 'Pará', 'PB': 'Paraíba', 'PR': 'Paraná', 'PE': 'Pernambuco', 'PI': 'Piauí', 'RJ': 'Rio de Janeiro', 'RN': 'Rio Grande do Norte', 'RS': 'Rio Grande do Sul', 'RO': 'Rondônia', 'RR': 'Roraima', 'SC': 'Santa Catarina', 'SP': 'São Paulo', 'SE': 'Sergipe', 'TO': 'Tocantins'
};

// Variável para armazenar os dados geográficos dos estados
const geojsonData = ref(null);

// Função auxiliar para calcular os limites (bounds) de uma geometria (Polygon ou MultiPolygon)
const getBounds = (geometry) => {
    let coords = [];
    if (geometry.type === 'Polygon') {
        coords = geometry.coordinates[0];
    } else if (geometry.type === 'MultiPolygon') {
        geometry.coordinates.forEach(poly => {
            poly[0].forEach(c => coords.push(c));
        });
    }
    
    let minLng = Infinity, minLat = Infinity, maxLng = -Infinity, maxLat = -Infinity;
    coords.forEach(([lng, lat]) => {
        if (lng < minLng) minLng = lng;
        if (lat < minLat) minLat = lat;
        if (lng > maxLng) maxLng = lng;
        if (lat > maxLat) maxLat = lat;
    });
    return [[minLng, minLat], [maxLng, maxLat]];
};

// --- CÁLCULOS (REAL vs SIMULADO) ---

/**
 * 1. VENDA ANUALIZADA REAL (O que você perguntou)
 * Baseado puramente nos campos do card amarelo: (Lts / Meses) * 12
 */
const vendaAnualizadaReal = computed(() => {
    if (!simuladorShare.suaVendaAtualLitros || !simuladorShare.mesesCorridos) return 0;
    return (simuladorShare.suaVendaAtualLitros / simuladorShare.mesesCorridos) * 12;
});

/**
 * 2. MARKET SHARE REAL
 * Comparação da Venda Anualizada Real vs Potencial Total
 */
const marketShareReal = computed(() => {
  if (!overviewData.value?.potencialConsumoLitrosAno || vendaAnualizadaReal.value <= 0) return '0.00';
  const ms = (vendaAnualizadaReal.value / overviewData.value.potencialConsumoLitrosAno) * 100;
  return ms.toFixed(2);
});

/**
 * 3. VENDA OBJETIVO (SIMULAÇÃO)
 * O quanto você venderia se atingisse o "Share Desejado"
 * Baseado em: Potencial Total * (Share Desejado / 100)
 */
const vendaObjetivoSimulada = computed(() => {
    const potencialTotal = overviewData.value?.potencialConsumoLitrosAno || 0;
    const shareAlvo = simuladorShare.shareDesejado || 0;
    return (potencialTotal * shareAlvo) / 100;
});

/**
 * 4. Aplica a simulação proporcional na tabela
 */
const aplicarSimulacaoNaTabela = () => {
    if (!tableData.value.length) return;
    const shareAlvo = parseFloat(simuladorShare.shareDesejado) || 0;
    
    tableData.value.forEach(row => {
        const vendaSimulada = (row.potencialLitros * shareAlvo) / 100;
        row.suaVendaEstimada = Math.round(vendaSimulada);
        row.gapLitros = Math.max(0, row.potencialLitros - row.suaVendaEstimada);
        row.share = shareAlvo.toFixed(2);
    });
};

const updateRowCalculations = (row) => {
    const venda = parseFloat(row.suaVendaEstimada) || 0;
    row.gapLitros = Math.max(0, row.potencialLitros - venda);
    row.share = row.potencialLitros > 0 ? ((venda / row.potencialLitros) * 100).toFixed(2) : '0.00';
};

// --- FUNÇÕES DE LÓGICA ---

const toggleVehicleType = (typeId) => {
    const index = filters.tipoVeiculo.indexOf(typeId);
    if (index > -1) {
        if (filters.tipoVeiculo.length > 1) filters.tipoVeiculo.splice(index, 1);
    } else {
        filters.tipoVeiculo.push(typeId);
    }
};

const getTrocaData = (tipoBusca) => {
    if (!simuladorTrocasData.value || !Array.isArray(simuladorTrocasData.value)) {
        return { veiculos: 0, litrosAno: 0, trocasPorAno: 0 };
    }
    const found = simuladorTrocasData.value.find(s => 
        s.tipoVeiculo.toUpperCase().includes(tipoBusca.toUpperCase())
    );
    return found || { veiculos: 0, litrosAno: 0, trocasPorAno: tipoBusca === 'LEVE' ? 1 : tipoBusca === 'PESADA' ? 3.8 : 3 };
};

const fetchDashboardData = async () => {
    isLoading.value = true;
    const ufParam = selectedUF.value === 'Todos' ? '' : selectedUF.value;
    const params = {
        ...filters,
        estado: ufParam,
        tipoVeiculo: filters.tipoVeiculo.join(','),
        suaVendaAtualLitros: simuladorShare.suaVendaAtualLitros,
        mesesCorridos: simuladorShare.mesesCorridos,
        shareDesejado: simuladorShare.shareDesejado
    };

    try {
        const [resOverview, resSimulador, resSpecs] = await Promise.all([
            api.get('/overview', { params }),
            api.get('/simulador', { params }),
            api.get('/especificacoes', { params })
        ]);

        if (resOverview.data.data) overviewData.value = resOverview.data.data;
        if (resSimulador.data.data) simuladorTrocasData.value = resSimulador.data.data;
        if (resSpecs.data.data) {
            tableData.value = resSpecs.data.data.itens || [];
            aplicarSimulacaoNaTabela();
        }
    } catch (e) { console.error(e); } finally { 
        isLoading.value = false;
        if (map.value) map.value.resize();
    }
};

// --- MAPA ---
const mapContainer = ref(null);
const map = ref(null);
const MAPBOX_TOKEN = import.meta.env.VITE_MAPBOX_TOKEN || 'SUA_CHAVE_AQUI';

const initMap = () => {
  mapboxgl.accessToken = MAPBOX_TOKEN;
  map.value = new mapboxgl.Map({
    container: mapContainer.value,
    style: 'mapbox://styles/mapbox/light-v11',
    center: [-52, -15], zoom: 2.8, attributionControl: false
  });

  map.value.on('load', async () => {
    const url = 'https://raw.githubusercontent.com/codeforamerica/click_that_hood/master/public/data/brazil-states.geojson';
    
    // Buscamos o dado para o mapa e para nossa variável local
    const response = await fetch(url);
    geojsonData.value = await response.json();

    map.value.addSource('states', { type: 'geojson', data: geojsonData.value, generateId: true });
    map.value.addLayer({ id: 'states-fill', type: 'fill', source: 'states', paint: { 'fill-color': '#e97332', 'fill-opacity': ['case', ['==', ['get', 'sigla'], selectedUF.value], 0.5, 0.1] }});
    map.value.addLayer({ id: 'states-borders', type: 'line', source: 'states', paint: { 'line-color': '#e97332', 'line-width': 1 }});
    
    map.value.on('click', 'states-fill', (e) => {
        const sigla = e.features[0].properties.sigla;
        selectedUF.value = (selectedUF.value === sigla) ? 'Todos' : sigla;
    });
  });
};

const updateMapHighlight = () => {
    if(!map.value || !map.value.isStyleLoaded()) return;
    
    // 1. Atualiza a cor (destaque)
    map.value.setPaintProperty('states-fill', 'fill-opacity', ['case', ['==', ['get', 'sigla'], selectedUF.value], 0.5, 0.1]);

    // 2. Move a câmera
    if (selectedUF.value !== 'Todos' && geojsonData.value) {
        // Encontra o estado selecionado no nosso GeoJSON
        const feature = geojsonData.value.features.find(f => f.properties.sigla === selectedUF.value);
        if (feature) {
            const bounds = getBounds(feature.geometry);
            map.value.fitBounds(bounds, {
                padding: 40,      // Espaçamento nas bordas do card
                duration: 1500,   // Velocidade da animação em milisegundos
                essential: true
            });
        }
    } else {
        // Se selecionar "Brasil Todo", volta para a visão geral do país
        map.value.flyTo({
            center: [-52, -15],
            zoom: 2.8,
            duration: 1500
        });
    }
};

// --- WATCHERS ---
watch(() => simuladorShare.shareDesejado, () => { aplicarSimulacaoNaTabela(); });
watch([selectedUF, () => filters.tipoVeiculo], () => { updateMapHighlight(); fetchDashboardData(); }, { deep: true });
watch([() => filters.viscosidade, () => filters.api, () => filters.acea, () => filters.jaso, () => filters.basico], () => { fetchDashboardData(); });

onMounted(async () => {
    initMap();
    const res = await api.get('/filtros').catch(() => null);
    if (res?.data?.data) apiFilters.value = res.data.data;
    await fetchDashboardData();
});

const fmtNum = (v) => v ? new Intl.NumberFormat('pt-BR').format(Math.floor(v)) : '0';
</script>

<template>
  <div class="dashboard-wrapper">
    
    <div class="top-filter-bar shadow-sm px-4 py-2 bg-white d-flex align-items-center gap-3">
      <div v-if="apiFilters" class="d-flex gap-3">
        <div v-for="f in [{l:'VISCOSIDADE', k:'viscosidade'}, {l:'API', k:'api'}, {l:'ACEA', k:'acea'}, {l:'JASO', k:'jaso'}, {l:'BÁSICO', k:'basico'}]" :key="f.k" class="filter-item">
            <label>{{ f.l }}:</label>
            <select v-model="filters[f.k]" class="form-select form-select-sm">
                <option value="Todos">Todos</option>
                <option v-for="opt in apiFilters[f.k]" :key="opt" :value="opt">{{ opt }}</option>
            </select>
        </div>
      </div>
      <button @click="filters.tipoVeiculo=['LEVE']; selectedUF='Todos'; Object.keys(filters).forEach(k => k !== 'tipoVeiculo' ? filters[k]='Todos' : null)" class="btn btn-red-total btn-sm ms-auto fw-bold px-4">Limpar todos os filtros</button>
    </div>

    <div class="main-content d-flex p-3 gap-3">
      
      <div class="sidebar d-flex flex-column gap-3">
        <div class="card border-0 shadow-sm p-3 rounded-4 bg-white">
            <h6 class="fw-bold mb-3 uppercase"><Filter :size="16" class="inline-block mr-1"/> Filtros</h6>
            <div class="mb-3">
                <label class="small fw-bold text-muted d-block mb-1">ESTADO</label>
                <select v-model="selectedUF" class="form-select form-select-sm">
                    <option value="Todos">Brasil Todo</option>
                    <option v-for="uf in apiFilters?.estados" :key="uf" :value="uf">{{ stateNames[uf] || uf }}</option>
                </select>
            </div>
            <div>
              <label class="small fw-bold text-muted d-block mb-1">TIPO DE VEÍCULO</label>
              <div class="btn-group w-100">
                  <button @click="toggleVehicleType('LEVE')" class="btn btn-vehicle-type" :class="{active: filters.tipoVeiculo.includes('LEVE')}">
                      <Car :size="18" /><span class="btn-label">Leve</span>
                  </button>
                  <button @click="toggleVehicleType('MOTO')" class="btn btn-vehicle-type" :class="{active: filters.tipoVeiculo.includes('MOTO')}">
                      <Bike :size="18" /><span class="btn-label">Moto</span>
                  </button>
                  <button @click="toggleVehicleType('PESADA')" class="btn btn-vehicle-type" :class="{active: filters.tipoVeiculo.includes('PESADA')}">
                      <Truck :size="18" /><span class="btn-label">Pesada</span>
                  </button>
              </div>
            </div>
        </div>

        <div class="card border-0 shadow-sm rounded-4 bg-white overflow-hidden" style="height: 250px;">
            <div ref="mapContainer" class="w-100 h-100"></div>
        </div>

        <!-- SIMULADOR DE SHARE (REAL) -->
        <div class="card border-0 shadow-sm p-3 rounded-4 simulator-card-total"> 
            <h6 class="fw-bold mb-2 uppercase" style="font-size: 11px;"><Calculator :size="16"/> Simulador de Share</h6>
            <div class="row g-2 mb-3 mt-2">
                <div class="col-8">
                    <label class="fw-bold small">SUA VENDA (Lts)</label>
                    <div class="pbi-input-custom">
                        <span class="pbi-input-label">Lts</span>
                        <input type="number" v-model="simuladorShare.suaVendaAtualLitros" class="pbi-input-field">
                    </div>
                </div>
                <div class="col-4">
                    <label class="fw-bold small">MESES</label>
                    <input type="number" v-model="simuladorShare.mesesCorridos" class="form-control form-control-sm fw-bold">
                </div>
            </div>
            
            <!-- EXIBIÇÃO DA VENDA ANUALIZADA REAL PARA NÃO PERDER A INFORMAÇÃO -->
            <div class="small fw-bold text-muted mb-2 text-center">
               Venda Anualizada Real: {{ fmtNum(vendaAnualizadaReal) }} Lts
            </div>

            <div class="d-flex align-items-center justify-content-between mb-3 bg-white p-2 rounded-3 border">
                <span class="fw-bold small">MARKET SHARE REAL:</span>
                <span class="fw-black text-orange fs-5">{{ marketShareReal }}%</span>
            </div>
            <label class="fw-bold small mb-1">SHARE DESEJADO (%)</label>
            <div class="bg-white border rounded p-1 d-flex align-items-center">
                <input type="number" v-model="simuladorShare.shareDesejado" class="form-control border-0 text-center fw-bold fs-4">
                <span class="fw-bold fs-5 pe-2">%</span>
            </div>
        </div>
      </div>

      <div class="content-grid flex-grow-1 d-flex flex-column gap-3">
        <div class="row g-3">
          <div class="col-lg-8">
            <div class="card border-0 shadow-sm p-3 rounded-4 bg-white h-100">
                <div class="px-2 mb-3">
                    <h6 class="fw-bold m-0 uppercase text-muted" style="font-size: 12px;">Simulador de Trocas Anual por Tipo de Veículo</h6>
                </div>
                <div class="row g-0 align-items-center">
                    <div class="col-4 px-4 border-end" :class="{'opacity-25': !filters.tipoVeiculo.includes('LEVE')}">
                        <div class="d-flex align-items-center gap-3 mb-4">
                            <Car class="text-orange" :size="48" />
                            <div class="lh-1">
                                <div class="fw-bold fs-4">{{ fmtNum(getTrocaData('LEVE').veiculos) }}</div>
                                <small class="text-muted fw-bold" style="font-size: 11px;">Veículos</small>
                            </div>
                        </div>
                        <div class="mb-4 pt-3 border-top">
                            <div class="fw-bold fs-4">{{ fmtNum(getTrocaData('LEVE').litrosAno) }}</div>
                            <small class="text-muted fw-bold" style="font-size: 11px;">Litros / ano</small>
                        </div>
                        <div class="p-2 rounded-3 border border-light-subtle text-center bg-white shadow-sm">
                            <small class="text-muted d-block mb-1 fw-bold" style="font-size:10px">Trocas por Ano</small>
                            <div class="d-flex align-items-center justify-content-center gap-2">
                                <span class="fw-bold">{{ getTrocaData('LEVE').trocasPorAno }}</span>
                                <Info :size="14" class="text-muted" />
                            </div>
                        </div>
                    </div>
                    <div class="col-4 px-4 border-end" :class="{'opacity-25': !filters.tipoVeiculo.includes('PESADA')}">
                        <div class="d-flex align-items-center gap-3 mb-4">
                            <Truck class="text-orange" :size="48" />
                            <div class="lh-1">
                                <div class="fw-bold fs-4">{{ fmtNum(getTrocaData('PESADA').veiculos) }}</div>
                                <small class="text-muted fw-bold" style="font-size: 11px;">Veículos</small>
                            </div>
                        </div>
                        <div class="mb-4 pt-3 border-top">
                            <div class="fw-bold fs-4">{{ fmtNum(getTrocaData('PESADA').litrosAno) }}</div>
                            <small class="text-muted fw-bold" style="font-size: 11px;">Litros / ano</small>
                        </div>
                        <div class="p-2 rounded-3 border border-light-subtle text-center bg-white shadow-sm">
                            <small class="text-muted d-block mb-1 fw-bold" style="font-size:10px">Trocas por Ano</small>
                            <div class="d-flex align-items-center justify-content-center gap-2">
                                <span class="fw-bold">{{ getTrocaData('PESADA').trocasPorAno }}</span>
                                <Info :size="14" class="text-muted" />
                            </div>
                        </div>
                    </div>
                    <div class="col-4 px-4" :class="{'opacity-25': !filters.tipoVeiculo.includes('MOTO')}">
                        <div class="d-flex align-items-center gap-3 mb-4">
                            <Bike class="text-orange" :size="48" />
                            <div class="lh-1">
                                <div class="fw-bold fs-4">{{ fmtNum(getTrocaData('MOTO').veiculos) }}</div>
                                <small class="text-muted fw-bold" style="font-size: 11px;">Veículos</small>
                            </div>
                        </div>
                        <div class="mb-4 pt-3 border-top">
                            <div class="fw-bold fs-4">{{ fmtNum(getTrocaData('MOTO').litrosAno) }}</div>
                            <small class="text-muted fw-bold" style="font-size: 11px;">Litros / ano</small>
                        </div>
                        <div class="p-2 rounded-3 border border-light-subtle text-center bg-white shadow-sm">
                            <small class="text-muted d-block mb-1 fw-bold" style="font-size:10px">Trocas por Ano</small>
                            <div class="d-flex align-items-center justify-content-center gap-2">
                                <span class="fw-bold">{{ getTrocaData('MOTO').trocasPorAno }}</span>
                                <Info :size="14" class="text-muted" />
                            </div>
                        </div>
                    </div>
                </div>
            </div>
          </div>

          <div class="col-lg-4">
            <div class="card border-0 shadow-sm rounded-4 overview-card text-white d-flex flex-column overflow-hidden h-100" style="background-color: #1a2332;">
              <div class="p-3 flex-grow-1">
                <p class="mb-1 fw-bold opacity-75 small">Potencial Consumo (L / Ano)</p>
                <h1 class="fw-bold m-0" style="font-size: 2.2rem;">{{ fmtNum(overviewData.potencialConsumoLitrosAno) }}</h1>
                <div class="mt-3">
                  <!-- PROJETADA BASEADA NO ALVO (SIMULADA) -->
                  <h3 class="fw-bold m-0">{{ fmtNum(vendaObjetivoSimulada) }}</h3>
                  <p class="mb-0 small opacity-75">Sua Venda Projetada (Alvo)</p>
                </div>
              </div>
              <div class="p-3" style="background-color: #e97332;">
                <h3 class="fw-bold m-0">{{ simuladorShare.shareDesejado }}%</h3>
                <p class="mb-3 small">Share Projetado (Alvo)</p>
                <h2 class="fw-bold m-0">{{ fmtNum(overviewData.frotaEfetiva) }}</h2>
                <p class="mb-0 small">Frota Eletiva</p>
              </div>
            </div>
          </div>
        </div>

        <div class="card border-0 shadow-sm rounded-4 bg-white overflow-hidden flex-grow-1">
            <div class="card-header bg-white border-0 py-3 px-4 d-flex justify-content-between align-items-center">
                <h6 class="fw-bold m-0 uppercase">DETALHAMENTO POR ESPECIFICAÇÃO</h6>
                <span class="badge rounded-pill bg-light text-dark border px-3">{{ tableData.length }} Registros</span>
            </div>
            <div class="table-responsive" style="max-height: 450px;">
                <table class="table table-hover align-middle mb-0">
                    <thead class="sticky-top bg-white border-bottom shadow-sm">
                        <tr>
                            <th class="ps-4 py-3 text-muted fw-bold small">ESPECIFICAÇÃO</th>
                            <th class="text-end py-3 text-muted fw-bold small">POTENCIAL (L)</th>
                            <th class="text-end py-3 text-orange fw-bold small" style="width: 180px;">SUA VENDA (EST.)</th>
                            <th class="text-end py-3 text-muted fw-bold small">GAP (LITROS)</th>
                            <th class="text-end pe-4 py-3 text-muted fw-bold small">SHARE</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(row, idx) in tableData" :key="idx" class="border-bottom">
                            <td class="ps-4 fw-bold text-dark small">{{ row.especificacao }}</td>
                            <td class="text-end fw-bold">{{ fmtNum(row.potencialLitros) }}</td>
                            <td class="text-end">
                                <div class="d-flex justify-content-end">
                                    <input type="number" v-model.number="row.suaVendaEstimada" @input="updateRowCalculations(row)" class="form-control form-control-sm text-end fw-bold text-orange border-orange-subtle bg-white" style="max-width: 110px;">
                                </div>
                            </td>
                            <td class="text-end text-muted">{{ fmtNum(row.gapLitros) }}</td>
                            <td class="text-end pe-4"><span class="badge bg-light text-dark border">{{ row.share }}%</span></td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
      </div>
    </div>
    <div v-if="isLoading" class="loader-global"><div class="spinner-border text-orange"></div></div>
  </div>
</template>

<style scoped>
.dashboard-wrapper { background-color: #f1f5f9; height: 100vh; display: flex; flex-direction: column; overflow: hidden; font-family: 'Inter', sans-serif; position: relative; }
.top-filter-bar { height: 60px; z-index: 100; border-bottom: 1px solid #e2e8f0; }
.filter-item { display: flex; align-items: center; gap: 8px; }
.filter-item label { font-size: 9px; font-weight: 800; color: #334155; white-space: nowrap; }
.sidebar { width: 300px; flex-shrink: 0; }
.content-grid { overflow-y: auto; padding-bottom: 60px; }
.btn-vehicle-type { border: 1px solid #dee2e6; background: #fff; color: #6c757d; flex: 1; display: flex; flex-direction: column; align-items: center; padding: 8px 0; transition: 0.2s; }
.btn-vehicle-type.active { background-color: #e97332 !important; border-color: #e97332 !important; color: white !important; }
.btn-label { font-size: 9px; font-weight: 700; margin-top: 4px; }
.simulator-card-total { background-color: #fff5e6; border: 1px solid #f1c40f !important; }
.pbi-input-custom { display: flex; align-items: center; background: white; border: 1px solid #dee2e6; border-radius: 50px; overflow: hidden; height: 32px; }
.pbi-input-label { background: #f8fafc; color: #1a2332; font-weight: 800; font-size: 10px; padding: 0 10px; border-right: 1px solid #dee2e6; }
.pbi-input-field { border: none; flex: 1; padding: 0 8px; font-size: 12px; font-weight: 700; outline: none; background: transparent; }
.loader-global { position: fixed; top:0; left:0; width:100%; height:100%; background:rgba(255,255,255,0.7); display:flex; align-items:center; justify-content:center; z-index: 9999; }
.uppercase { text-transform: uppercase; letter-spacing: 0.5px; }
.text-orange { color: #e97332 !important; }
.border-orange-subtle { border-color: #fbd6c0 !important; }
.btn-red-total { background-color: #ff0000; color: white; border: none; font-size: 11px; }
input::-webkit-outer-spin-button, input::-webkit-inner-spin-button { -webkit-appearance: none; margin: 0; }
</style>
