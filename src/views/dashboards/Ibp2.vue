<script setup>
import { computed, onMounted, reactive, ref, watch } from 'vue';

const API_BASE = 'https://lubx-api.lubconsulta.com.br/bi/ipb';
const DEFAULT_SEGMENT = 'REVENDEDOR';
const ALL_BRANDS_VALUE = 'TODAS';
const SHARE_TICKS = [1, 2, 3, 4, 5, 10, 15, 20, 25, 100];

const loading = ref(true);
const dashboardData = ref(null);

const state = reactive({
  empresaSelecionada: ALL_BRANDS_VALUE,
  segmentos: [DEFAULT_SEGMENT],
  dataDe: '',
  dataAte: '',
  hiddenEmpresas: [],
});

const marcasConfig = {
  TOTALENERGIES: { cor: '#f58220', logo: 'totalenergies' },
  YPF: { cor: '#00529b', logo: 'ypf' },
  VIBRA: { cor: '#004415', logo: 'vibra' },
  PETRONAS: { cor: '#00A79D', logo: 'petronas' },
  CASTROL: { cor: '#E10600', logo: 'castrol' },
  VALVOLINE: { cor: '#2FA4E7', logo: 'valvoline' },
  'RAIZEN LUBRIFICANTES': { cor: '#9B1BB3', logo: 'raizen' },
  RAIZEN: { cor: '#9B1BB3', logo: 'raizen' },
  MOOVE: { cor: '#00A3E0', logo: 'moove' },
  LWART: { cor: '#000000', logo: 'lwart' },
  ICONIC: { cor: '#F56600', logo: 'iconic' },
};

const availableEmpresas = computed(() => {
  const list = dashboardData.value?.filtros?.empresas || [];
  return list.filter(e => e !== 'IPIRANGA' && e !== 'CHEVRON');
});

const availableSegmentos = computed(() => dashboardData.value?.filtros?.segmentos || []);

const selectedEmpresaLabel = computed(() => (
  state.empresaSelecionada === ALL_BRANDS_VALUE ? 'Todas as marcas' : state.empresaSelecionada
));

function formatDateToISO(date) {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, '0');
  const d = String(date.getDate()).padStart(2, '0');
  return `${y}-${m}-${d}`;
}

function initializeFiltersFromResponse(data) {
  const filtros = data?.filtros || {};
  const empresas = filtros.empresas || [];
  const segmentos = filtros.segmentosSelecionados || [];

  if (!state.dataDe || !state.dataAte) {
    if (filtros.ultimaCompetencia && filtros.ultimaCompetencia.ano && filtros.ultimaCompetencia.mes) {
      const ano = parseInt(filtros.ultimaCompetencia.ano);
      const mes = parseInt(filtros.ultimaCompetencia.mes);
      if (!isNaN(ano) && !isNaN(mes)) {
        const dataAteObj = new Date(ano, mes, 0);
        const dataDeObj = new Date(ano - 1, mes - 1, 1);
        state.dataAte = formatDateToISO(dataAteObj);
        state.dataDe = formatDateToISO(dataDeObj);
      }
    }
  }

  if (!state.segmentos.length) state.segmentos = segmentos.length ? [...segmentos] : [DEFAULT_SEGMENT];
  if (!empresas.includes(state.empresaSelecionada) && state.empresaSelecionada !== ALL_BRANDS_VALUE) {
    state.empresaSelecionada = filtros.empresaSelecionada || ALL_BRANDS_VALUE;
  }
}

async function fetchDashboard() {
  loading.value = true;
  try {
    const params = new URLSearchParams();
    params.append('empresa', state.empresaSelecionada);
    params.append('dataDe', state.dataDe);
    params.append('dataAte', state.dataAte);
    state.segmentos.forEach((segmento) => params.append('segmento', segmento));

    const response = await fetch(`${API_BASE}/dashboard?${params.toString()}`);
    const json = await response.json();

    if (json.status === 'ok') {
      initializeFiltersFromResponse(json.data);
      dashboardData.value = json.data;
    }
  } catch (error) {
    console.error('Erro ao buscar dados:', error);
  } finally {
    loading.value = false;
  }
}

function resetFilters() {
  state.empresaSelecionada = ALL_BRANDS_VALUE;
  state.segmentos = [DEFAULT_SEGMENT];
  state.hiddenEmpresas = [];
  state.dataDe = '';
  state.dataAte = '';
  fetchDashboard();
}

function toggleSegmento(segmentoId) {
  const alreadySelected = state.segmentos.includes(segmentoId);
  if (alreadySelected && state.segmentos.length === 1) return;
  state.segmentos = alreadySelected ? state.segmentos.filter(i => i !== segmentoId) : [...state.segmentos, segmentoId];
}

function toggleLineVisibility(empresa) {
  const alreadyHidden = state.hiddenEmpresas.includes(empresa);
  const visibleCount = availableEmpresas.value.length - state.hiddenEmpresas.length;
  if (!alreadyHidden && visibleCount <= 1) return;
  state.hiddenEmpresas = alreadyHidden ? state.hiddenEmpresas.filter(i => i !== empresa) : [...state.hiddenEmpresas, empresa];
}

function formatDateDisplay(dateStr) {
  if (!dateStr) return '';
  const [year, month] = dateStr.split('-');
  return `${month}/${year}`;
}

function formatVolume(value) {
  const numeric = Number(value || 0);
  return `${numeric.toLocaleString('pt-BR', { maximumFractionDigits: 3 })} m³`;
}

function formatVolumeCompact(value) {
  const numeric = Number(value || 0);
  if (numeric >= 1000) return `${(numeric / 1000).toLocaleString('pt-BR', { maximumFractionDigits: 1 })} mil m³`;
  return `${numeric.toLocaleString('pt-BR', { maximumFractionDigits: 1 })} m³`;
}

function formatVolumeAnualMil(value) {
  return (Number(value || 0) / 1000).toLocaleString('pt-BR', { minimumFractionDigits: 0, maximumFractionDigits: 1 });
}

function formatPercent(value) {
  return `${Number(value || 0).toLocaleString('pt-BR', { minimumFractionDigits: 1, maximumFractionDigits: 1 })}%`;
}

function getShareY(share, chartBottom = 286, chartHeight = 230) {
  const safeShare = Math.max(0, Math.min(100, Number(share || 0)));
  const levels = [0, ...SHARE_TICKS];
  const totalSegments = levels.length - 1;
  for (let index = 0; index < totalSegments; index += 1) {
    const start = levels[index];
    const end = levels[index + 1];
    if (safeShare <= end || index === totalSegments - 1) {
      const fraction = end > start ? (safeShare - start) / (end - start) : 0;
      const position = index + Math.max(0, Math.min(1, fraction));
      return chartBottom - ((position / totalSegments) * chartHeight);
    }
  }
  return chartBottom;
}

const chartMetrics = computed(() => {
  const chart = dashboardData.value?.graficos?.vendasPorMes;
  if (!chart?.series?.length) return null;

  const allMeses = chart.meses || [];
  const seriesRaw = chart.series || [];
  const marketRaw = chart.totaisMercado?.length 
    ? chart.totaisMercado.map(v => Number(v || 0))
    : allMeses.map((_, i) => seriesRaw.reduce((sum, s) => sum + Number(s.valores?.[i] || 0), 0));

  // --- LÓGICA DE FILTRAGEM DE MESES (Ponto 4) ---
  const monthMap = { Jan: 1, Fev: 2, Mar: 3, Abr: 4, Mai: 5, Jun: 6, Jul: 7, Ago: 8, Set: 9, Out: 10, Nov: 11, Dez: 12 };
  
  // Converte filtros para numero comparável YYYYMM
  const startLimit = parseInt(state.dataDe.substring(0, 7).replace('-', '')); 
  const endLimit = parseInt(state.dataAte.substring(0, 7).replace('-', ''));

  // Filtra os índices que estão no range
  const validIndices = [];
  allMeses.forEach((m, idx) => {
    const [mesStr, anoStr] = m.split('/');
    const yyyymm = parseInt(`20${anoStr}${String(monthMap[mesStr]).padStart(2, '0')}`);
    if (yyyymm >= startLimit && yyyymm <= endLimit) {
      validIndices.push(idx);
    }
  });

  // Se o filtro resultou em vazio (ex: seleção futura), mostramos o que a API mandou para não quebrar
  const targetIndices = validIndices.length > 0 ? validIndices : allMeses.map((_, i) => i);

  const meses = targetIndices.map(i => allMeses[i]);
  const marketVolumes = targetIndices.map(i => marketRaw[i]);

  const startX = 84;
  const chartWidth = 850;
  const chartBottom = 286;
  const chartHeight = 230;
  const gap = meses.length > 1 ? chartWidth / (meses.length - 1) : 0;
  const maxMarketVolume = Math.max(...marketVolumes, 1);
  const highlightedCompany = state.empresaSelecionada === ALL_BRANDS_VALUE ? null : state.empresaSelecionada;

  const allLines = seriesRaw
    .filter(s => !state.hiddenEmpresas.includes(s.empresa) && s.empresa !== 'IPIRANGA' && s.empresa !== 'CHEVRON')
    .map((serie) => {
      const isHighlighted = highlightedCompany === serie.empresa;
      return {
        empresa: serie.empresa,
        cor: marcasConfig[serie.empresa]?.cor || '#8c8c8c',
        isHighlighted,
        strokeWidth: highlightedCompany ? (isHighlighted ? 3.6 : 1.8) : 2.1,
        strokeOpacity: highlightedCompany ? (isHighlighted ? 1 : 0.45) : 0.82,
        points: targetIndices.map((originalIdx, localIdx) => {
          const volume = Number(serie.valores?.[originalIdx] || 0);
          const marketVolume = Number(marketRaw[originalIdx] || 0);
          const share = marketVolume > 0 ? (volume / marketVolume) * 100 : 0;
          return {
            x: startX + (gap * localIdx),
            y: getShareY(share, chartBottom, chartHeight),
            shareLabel: formatPercent(share),
            volumeLabel: formatVolume(volume),
            mes: allMeses[originalIdx]
          };
        }),
      };
    });

  const bars = marketVolumes.map((volume, index) => {
    const height = (volume / maxMarketVolume) * 170;
    return {
      x: startX + (gap * index) - 18,
      y: chartBottom - height,
      height,
      width: 36,
      value: volume
    };
  });

  const sidebarShares = (dashboardData.value?.marcas || [])
    .filter(m => m.nome !== 'IPIRANGA' && m.nome !== 'CHEVRON')
    .sort((a, b) => b.share - a.share);

  return {
    meses, bars, allLines, sidebarShares, startX, gap, chartBottom,
    yTicks: SHARE_TICKS.map(tick => ({ value: tick, y: getShareY(tick, chartBottom, chartHeight) })),
    latestShareMonth: meses[meses.length - 1] || ''
  };
});

const annualLinePoints = computed(() => {
  const annual = dashboardData.value?.graficos?.vendaAnual;
  if (!annual?.items || state.empresaSelecionada === ALL_BRANDS_VALUE) return null;
  const items = annual.items;
  const maxVal = Math.max(...items.map(i => i.valorTotal || i.valor), 1);
  return items.map((item, index) => {
    const xPercent = (index / (items.length - 1)) * 100;
    const val = item.valorMarca || item.valor;
    const yPercent = 100 - ((val / maxVal) * 100);
    return { x: xPercent, y: yPercent };
  });
});

watch(
  () => [state.empresaSelecionada, state.dataDe, state.dataAte, state.segmentos.join('|')],
  () => {
    if (!state.dataDe || !state.dataAte || !state.segmentos.length) return;
    fetchDashboard();
  }
);

onMounted(fetchDashboard);
</script>

<template>
  <div class="pbi-layout" v-if="dashboardData">
    <header class="pbi-filters">
      <div class="filter-left">
        <div class="filter-block">
          <label>Selecione sua marca</label>
          <select v-model="state.empresaSelecionada" class="select-control">
            <option :value="ALL_BRANDS_VALUE">Todas as marcas</option>
            <option v-for="empresa in availableEmpresas" :key="empresa" :value="empresa">{{ empresa }}</option>
          </select>
        </div>
        <div class="filter-block">
          <label>Segmento</label>
          <div class="segment-picker">
            <label v-for="segmento in availableSegmentos" :key="segmento.id" class="segment-option">
              <input type="checkbox" :checked="state.segmentos.includes(segmento.id)" @change="toggleSegmento(segmento.id)" />
              <span>{{ segmento.label }}</span>
            </label>
          </div>
        </div>
        <div class="filter-block">
          <label>Período</label>
          <div class="period-picker">
            <div class="date-field"><span>De</span><input type="date" v-model="state.dataDe" /></div>
            <div class="date-field"><span>Até</span><input type="date" v-model="state.dataAte" /></div>
          </div>
        </div>
      </div>
      <button class="btn-clear" @click="resetFilters">Limpar filtros</button>
    </header>

    <div class="pbi-content" :class="{ 'loading-opacity': loading }">
      <section class="top-section">
        <div class="kpi-grid">
          <div v-for="(card, index) in dashboardData.cards" :key="index" class="card kpi-card" :style="{ background: card.id.includes('empresa') ? '#fff6ed' : '#fff' }">
            <div class="kpi-label">{{ card.titulo }}</div>
            <div class="kpi-val">{{ formatVolume(card.valor) }}</div>
            <div class="kpi-footer">
              <span class="sub">{{ formatVolume(card.comparativo) }}</span>
              <span class="pill" :class="{ neg: card.variacaoPercentual < 0 }">{{ formatPercent(card.variacaoPercentual) }} YoY</span>
            </div>
          </div>
        </div>

        <div class="card chart-annual">
          <div class="card-title">Vendas Anual (Mil por metros Cúbicos)</div>
          <div class="bars-container" style="position: relative;">
            <div v-for="item in dashboardData.graficos.vendaAnual.items" :key="item.ano" class="bar-col">
              <span class="bar-txt">{{ formatVolumeAnualMil(item.valorTotal || item.valor) }}</span>
              <div class="bar" :class="{ orange: state.empresaSelecionada === ALL_BRANDS_VALUE }" :style="{ height: `${((item.valorTotal || item.valor) / Math.max(...dashboardData.graficos.vendaAnual.items.map(e => e.valorTotal || e.valor), 1)) * 100}%` }"></div>
              <span class="bar-label">{{ item.ano }}</span>
            </div>
            <svg v-if="annualLinePoints" class="annual-line-overlay" viewBox="0 0 100 100" preserveAspectRatio="none">
              <polyline :points="annualLinePoints.map(p => `${p.x},${p.y}`).join(' ')" fill="none" :stroke="marcasConfig[state.empresaSelecionada]?.cor || '#f58220'" stroke-width="3" />
            </svg>
          </div>
        </div>

        <div class="card chart-segment">
          <div class="card-title">Vendas por segmento</div>
          <div class="segment-container">
            <div v-for="seg in dashboardData.graficos.segmento.items" :key="seg.id" class="seg-col">
              <span class="bar-txt">{{ formatVolumeCompact(seg.valor) }}</span>
              <div class="seg-bar" :style="{ background: seg.cor, height: '70%' }"></div>
              <span class="bar-label bold">{{ seg.label }}</span>
            </div>
          </div>
        </div>
      </section>

      <section class="card main-viz">
        <div class="viz-header">
          <div class="viz-info">
            <h3>Vendas de Lubrificantes por Mês</h3>
            <p>Período: {{ formatDateDisplay(state.dataDe) }} a {{ formatDateDisplay(state.dataAte) }}</p>
          </div>
        </div>

        <div class="viz-legend">
          <button v-for="empresa in availableEmpresas" :key="empresa" class="leg-item" :class="{ hidden: state.hiddenEmpresas.includes(empresa), active: state.empresaSelecionada === empresa }" @click="toggleLineVisibility(empresa)">
            <span class="dot" :style="{ background: marcasConfig[empresa]?.cor || '#8c8c8c' }"></span>
            <span>{{ empresa }}</span>
          </button>
        </div>

        <div class="viz-body">
          <div class="svg-area">
            <svg viewBox="0 0 1000 330" preserveAspectRatio="none">
              <g v-for="tick in chartMetrics?.yTicks" :key="tick.value">
                <text x="44" :y="tick.y + 4" text-anchor="end" class="svg-txt-axis">{{ tick.value }}%</text>
                <line x1="56" :y1="tick.y" x2="950" :y2="tick.y" stroke="#f2f3f5" />
              </g>
              <g v-for="(bar, index) in chartMetrics?.bars" :key="index">
                <rect :x="bar.x" :y="bar.y" :width="bar.width" :height="bar.height" fill="#e5e7eb" rx="4" />
              </g>
              <g v-for="line in chartMetrics?.allLines" :key="line.empresa">
                <polyline :points="line.points.map(p => `${p.x},${p.y}`).join(' ')" fill="none" :stroke="line.cor" :stroke-width="line.strokeWidth" :stroke-opacity="line.strokeOpacity" />
                <circle v-for="p in line.points" :key="p.mes" :cx="p.x" :cy="p.y" :r="line.isHighlighted ? 4.8 : 3.5" :fill="line.cor" />
              </g>
              <text v-for="(mes, i) in chartMetrics?.meses" :key="mes" :x="(chartMetrics?.startX || 84) + ((chartMetrics?.gap || 0) * i)" y="314" text-anchor="middle" class="svg-txt">{{ mes }}</text>
            </svg>
          </div>
          <aside class="sidebar">
            <div class="side-title">% Share Mercado</div>
            <div v-for="marca in chartMetrics?.sidebarShares" :key="marca.nome" class="side-item">
              <img :src="`/logos/lubrificantes/${marcasConfig[marca.nome]?.logo || 'generic'}.png`" class="brand-logo-img" />
              <span class="name" :class="{ 'bold-txt': state.empresaSelecionada === marca.nome }">{{ marca.nome }}</span>
              <span class="badge" :class="{ orange: state.empresaSelecionada === marca.nome }">{{ formatPercent(marca.share) }}</span>
            </div>
          </aside>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
/* Estilos mantidos conforme versão anterior */
.annual-line-overlay { position: absolute; top: 15%; left: 30px; right: 30px; height: 60%; width: calc(100% - 60px); pointer-events: none; }
.pbi-layout { background: #f3f3f3; min-height: 100vh; padding: 15px; font-family: 'Segoe UI', sans-serif; }
.pbi-content { display: flex; flex-direction: column; gap: 15px; min-width: 1200px; }
.loading-opacity { opacity: 0.5; pointer-events: none; }
.pbi-filters { display: flex; justify-content: space-between; align-items: center; background: white; padding: 10px 20px; border-radius: 8px; margin-bottom: 10px; }
.filter-left { display: flex; align-items: flex-end; gap: 16px; }
.filter-block { display: flex; flex-direction: column; gap: 6px; }
.filter-block label { font-size: 11px; font-weight: 700; color: #6b7280; text-transform: uppercase; }
.select-control { min-width: 200px; border: 1px solid #d1d5db; border-radius: 6px; padding: 8px; font-size: 12px; }
.segment-picker { display: flex; gap: 8px; }
.segment-option { display: inline-flex; align-items: center; gap: 4px; border: 1px solid #d1d5db; border-radius: 20px; padding: 4px 12px; font-size: 11px; background: #fafafa; }
.period-picker { display: flex; gap: 10px; background: #f8f9fa; padding: 5px 10px; border-radius: 8px; border: 1px solid #ddd; }
.date-field { display: flex; align-items: center; gap: 5px; font-size: 11px; }
.date-field input { border: 1px solid #ccc; border-radius: 4px; padding: 3px; }
.btn-clear { background: white; border: 1px solid #333; padding: 8px 15px; border-radius: 6px; font-weight: 700; cursor: pointer; font-size: 11px; }
.top-section { display: grid; grid-template-columns: 3fr 1.5fr 1fr; gap: 15px; height: 260px; }
.kpi-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; }
.card { background: white; border-radius: 10px; border: 1px solid #e0e0e0; padding: 12px; display: flex; flex-direction: column; }
.kpi-label, .card-title { font-size: 12px; font-weight: 700; color: #6b7280; }
.kpi-val { font-size: 18px; font-weight: 800; margin: 8px 0; }
.kpi-footer { display: flex; justify-content: space-between; margin-top: auto; }
.sub { font-size: 11px; color: #6b7280; }
.pill { background: #e6f7ed; color: #008a3e; font-size: 10px; font-weight: 700; padding: 2px 8px; border-radius: 4px; }
.pill.neg { background: #fee2e2; color: #dc2626; }
.bars-container, .segment-container { display: flex; align-items: flex-end; justify-content: space-between; flex: 1; gap: 8px; margin-top: 10px; }
.bar-col, .seg-col { display: flex; flex-direction: column; align-items: center; flex: 1; height: 100%; justify-content: flex-end; }
.bar { width: 60%; background: #cfd4d9; border-radius: 3px 3px 0 0; }
.bar.orange { background: #f58220; }
.seg-bar { width: 30px; border-radius: 3px 3px 0 0; }
.bar-txt { font-size: 9px; font-weight: 700; margin-bottom: 4px; }
.bar-label { font-size: 10px; color: #999; margin-top: 5px; }
.main-viz { min-height: 500px; margin-top: 15px; }
.viz-legend { display: flex; flex-wrap: wrap; gap: 10px; margin: 15px 0; padding-bottom: 10px; border-bottom: 1px solid #f0f0f0; }
.leg-item { display: inline-flex; align-items: center; gap: 6px; padding: 4px 12px; border: 1px solid #e5e7eb; border-radius: 20px; background: white; cursor: pointer; font-size: 11px; }
.leg-item.active { border-color: #f58220; color: #f58220; font-weight: 700; }
.dot { width: 12px; height: 3px; border-radius: 2px; }
.viz-body { display: flex; gap: 20px; flex: 1; }
.svg-area { flex: 1; }
.sidebar { width: 240px; border-left: 1px solid #eee; padding-left: 15px; }
.side-item { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.brand-logo-img { width: 20px; height: 20px; object-fit: contain; }
.name { flex: 1; font-size: 11px; }
.badge { font-size: 10px; background: #eee; padding: 2px 6px; border-radius: 4px; min-width: 50px; text-align: center; }
.badge.orange { background: #f58220; color: white; }
.svg-txt { font-size: 11px; fill: #9ca3af; }
.svg-txt-axis { font-size: 10px; fill: #9ca3af; font-weight: 700; }
</style>
