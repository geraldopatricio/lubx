<script setup>
import { computed, onMounted, reactive, ref, watch } from 'vue';

const API_BASE = 'https://lubx-api.lubconsulta.com.br/bi/ipb';
const DEFAULT_SEGMENT = 'REVENDEDOR';
const ALL_BRANDS_VALUE = 'TODAS';
const SHARE_TICKS = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 15, 20, 25, 100];

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
  CHEVRON: { cor: '#005da4', logo: 'chevron' },
  IPIRANGA: { cor: '#ffcb05', logo: 'ipiranga' },
};

const availableEmpresas = computed(() => dashboardData.value?.filtros?.empresas || []);
const availableSegmentos = computed(() => dashboardData.value?.filtros?.segmentos || []);

const selectedEmpresaLabel = computed(() => (
  state.empresaSelecionada === ALL_BRANDS_VALUE ? 'Todas as marcas' : state.empresaSelecionada
));

const visibleEmpresas = computed(() => {
  if (!dashboardData.value?.graficos?.vendasPorMes?.series) return [];
  return dashboardData.value.graficos.vendasPorMes.series
    .filter((serie) => !state.hiddenEmpresas.includes(serie.empresa))
    .map((serie) => serie.empresa);
});

function initializeFiltersFromResponse(data) {
  const filtros = data?.filtros || {};
  const empresas = filtros.empresas || [];
  const segmentos = filtros.segmentosSelecionados || [];

  if (!state.dataDe) state.dataDe = filtros.dataDeSelecionada || '';
  if (!state.dataAte) state.dataAte = filtros.dataAteSelecionada || '';
  if (!state.segmentos.length) state.segmentos = segmentos.length ? [...segmentos] : [DEFAULT_SEGMENT];

  if (!empresas.includes(state.empresaSelecionada) && state.empresaSelecionada !== ALL_BRANDS_VALUE) {
    state.empresaSelecionada = filtros.empresaSelecionada || ALL_BRANDS_VALUE;
  }

  if (!dashboardData.value) {
    state.empresaSelecionada = filtros.empresaSelecionada || ALL_BRANDS_VALUE;
    state.segmentos = segmentos.length ? [...segmentos] : [DEFAULT_SEGMENT];
    state.dataDe = filtros.dataDeSelecionada || state.dataDe;
    state.dataAte = filtros.dataAteSelecionada || state.dataAte;
  }

  state.hiddenEmpresas = state.hiddenEmpresas.filter((empresa) => empresas.includes(empresa));
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

  if (alreadySelected && state.segmentos.length === 1) {
    return;
  }

  state.segmentos = alreadySelected
    ? state.segmentos.filter((item) => item !== segmentoId)
    : [...state.segmentos, segmentoId];
}

function toggleLineVisibility(empresa) {
  const alreadyHidden = state.hiddenEmpresas.includes(empresa);
  const visibleCount = availableEmpresas.value.length - state.hiddenEmpresas.length;

  if (!alreadyHidden && visibleCount <= 1) {
    return;
  }

  state.hiddenEmpresas = alreadyHidden
    ? state.hiddenEmpresas.filter((item) => item !== empresa)
    : [...state.hiddenEmpresas, empresa];
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
  if (numeric >= 1000) {
    return `${(numeric / 1000).toLocaleString('pt-BR', { minimumFractionDigits: 0, maximumFractionDigits: 1 })} mil m³`;
  }
  return `${numeric.toLocaleString('pt-BR', { maximumFractionDigits: 1 })} m³`;
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

  const meses = chart.meses || [];
  const series = chart.series || [];
  const marketVolumes = chart.totaisMercado?.length
    ? chart.totaisMercado.map((item) => Number(item || 0))
    : meses.map((_, index) => series.reduce((sum, serie) => sum + Number(serie.valores?.[index] || 0), 0));

  const startX = 84;
  const chartWidth = 850;
  const chartBottom = 286;
  const chartHeight = 230;
  const gap = meses.length > 1 ? chartWidth / (meses.length - 1) : 0;
  const maxMarketVolume = Math.max(...marketVolumes, 1);
  const highlightedCompany = state.empresaSelecionada === ALL_BRANDS_VALUE ? null : state.empresaSelecionada;

  const allLines = series
    .filter((serie) => !state.hiddenEmpresas.includes(serie.empresa))
    .map((serie) => {
      const isHighlighted = highlightedCompany === serie.empresa;

      return {
        empresa: serie.empresa,
        cor: marcasConfig[serie.empresa]?.cor || '#8c8c8c',
        isHighlighted,
        strokeWidth: highlightedCompany ? (isHighlighted ? 3.6 : 1.8) : 2.1,
        strokeOpacity: highlightedCompany ? (isHighlighted ? 1 : 0.45) : 0.82,
        points: meses.map((mes, index) => {
          const volume = Number(serie.valores?.[index] || 0);
          const marketVolume = Number(marketVolumes[index] || 0);
          const share = marketVolume > 0 ? (volume / marketVolume) * 100 : 0;

          return {
            mes,
            volume,
            share,
            shareLabel: formatPercent(share),
            volumeLabel: formatVolume(volume),
            x: startX + (gap * index),
            y: getShareY(share, chartBottom, chartHeight),
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
      value: volume,
      label: formatVolumeCompact(volume),
    };
  });

  const sidebarShares = (dashboardData.value?.marcas || [])
    .map((marca) => {
      const serie = series.find((item) => item.empresa === marca.nome);
      const lastValue = Number(serie?.valores?.[serie.valores.length - 1] || 0);
      const lastMarket = Number(marketVolumes[marketVolumes.length - 1] || 0);
      const share = lastMarket > 0 ? (lastValue / lastMarket) * 100 : Number(marca.share || 0);

      return {
        ...marca,
        share,
      };
    })
    .sort((a, b) => b.share - a.share);

  return {
    meses,
    bars,
    allLines,
    sidebarShares,
    startX,
    gap,
    chartBottom,
    yTicks: SHARE_TICKS.map((tick) => ({ value: tick, y: getShareY(tick, chartBottom, chartHeight) })),
    latestShareMonth: meses[meses.length - 1] || '',
  };
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
              <input
                type="checkbox"
                :checked="state.segmentos.includes(segmento.id)"
                @change="toggleSegmento(segmento.id)"
              />
              <span>{{ segmento.label }}</span>
            </label>
          </div>
        </div>

        <div class="filter-block">
          <label>Período</label>
          <div class="period-picker">
            <div class="date-field">
              <span>De</span>
              <input type="date" v-model="state.dataDe" :max="state.dataAte || undefined" />
            </div>
            <div class="date-field">
              <span>Até</span>
              <input type="date" v-model="state.dataAte" :min="state.dataDe || undefined" />
            </div>
          </div>
        </div>
      </div>

      <button class="btn-clear" @click="resetFilters">Limpar filtros</button>
    </header>

    <div class="pbi-content" :class="{ 'loading-opacity': loading }">
      <section class="top-section">
        <div class="kpi-grid">
          <div
            v-for="(card, index) in dashboardData.cards"
            :key="index"
            class="card kpi-card"
            :style="{ background: card.id.includes('empresa') ? '#fff6ed' : '#fff' }"
          >
            <div class="kpi-label">{{ card.titulo }}</div>
            <div class="kpi-val">{{ formatVolume(card.valor) }}</div>
            <div class="kpi-footer">
              <span class="sub">{{ formatVolume(card.comparativo) }}</span>
              <span class="pill" :class="{ neg: card.variacaoPercentual < 0 }">
                {{ formatPercent(card.variacaoPercentual) }} YoY
              </span>
            </div>
          </div>
        </div>

        <div class="card chart-annual">
          <div class="card-title">Venda anual</div>
          <div class="bars-container">
            <div v-for="item in dashboardData.graficos.vendaAnual.items" :key="item.ano" class="bar-col">
              <span class="bar-txt">{{ formatVolumeCompact(item.valor) }}</span>
              <div
                class="bar"
                :class="{ orange: String(item.ano) === dashboardData.filtros.anoSelecionado }"
                :title="`${item.ano}: ${formatVolume(item.valor)}`"
                :style="{ height: `${(item.valor / Math.max(...dashboardData.graficos.vendaAnual.items.map((entry) => entry.valor), 1)) * 100}%` }"
              ></div>
              <span class="bar-label">{{ item.ano }}</span>
            </div>
          </div>
        </div>

        <div class="card chart-segment">
          <div class="card-title">Vendas por segmento</div>
          <div class="segment-container">
            <div v-for="seg in dashboardData.graficos.segmento.items" :key="seg.id" class="seg-col">
              <span class="bar-txt">{{ formatVolumeCompact(seg.valor) }}</span>
              <div class="seg-bar" :title="`${seg.label}: ${formatVolume(seg.valor)}`" :style="{ background: seg.cor, height: '70%' }"></div>
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
            <span v-if="dashboardData.filtros.ultimaCompetencia" class="caption">
              Última competência do IPB: {{ dashboardData.filtros.ultimaCompetencia.mes }}/{{ dashboardData.filtros.ultimaCompetencia.ano }}
            </span>
          </div>
        </div>

        <div class="viz-legend">
          <button
            v-for="empresa in availableEmpresas"
            :key="empresa"
            type="button"
            class="leg-item"
            :class="{
              hidden: state.hiddenEmpresas.includes(empresa),
              active: state.empresaSelecionada === empresa,
            }"
            @click="toggleLineVisibility(empresa)"
          >
            <span class="dot" :style="{ background: marcasConfig[empresa]?.cor || '#8c8c8c' }"></span>
            <span>{{ empresa }}</span>
          </button>
        </div>

        <div class="viz-body">
          <div class="svg-area">
            <svg viewBox="0 0 1000 330" preserveAspectRatio="none">
              <g class="axis-y">
                <line x1="56" :y1="chartMetrics?.chartBottom || 286" x2="950" :y2="chartMetrics?.chartBottom || 286" stroke="#eef0f2" stroke-width="1" />
                <g v-for="tick in chartMetrics?.yTicks" :key="tick.value">
                  <text x="44" :y="tick.y + 4" text-anchor="end" class="svg-txt-axis">{{ tick.value }}%</text>
                  <line x1="56" :y1="tick.y" x2="950" :y2="tick.y" stroke="#f2f3f5" stroke-width="1" />
                </g>
              </g>

              <g v-for="(bar, index) in chartMetrics?.bars" :key="`bar-${index}`">
                <rect :x="bar.x" :y="bar.y" :width="bar.width" :height="bar.height" fill="#e5e7eb" rx="4">
                  <title>{{ `Mercado total ${chartMetrics?.meses[index]}: ${formatVolume(bar.value)}` }}</title>
                </rect>
              </g>

              <g v-for="line in chartMetrics?.allLines" :key="line.empresa">
                <polyline
                  :points="line.points.map((point) => `${point.x},${point.y}`).join(' ')"
                  fill="none"
                  :stroke="line.cor"
                  :stroke-width="line.strokeWidth"
                  :stroke-opacity="line.strokeOpacity"
                  stroke-linejoin="round"
                  stroke-linecap="round"
                >
                  <title>{{ line.empresa }}</title>
                </polyline>

                <g v-for="point in line.points" :key="`${line.empresa}-${point.mes}`">
                  <circle :cx="point.x" :cy="point.y" :r="line.isHighlighted ? 4.8 : 3.5" :fill="line.cor">
                    <title>{{ `${line.empresa} | ${point.mes} | Share ${point.shareLabel} | Volume ${point.volumeLabel}` }}</title>
                  </circle>
                  <text
                    v-if="line.isHighlighted"
                    :x="point.x"
                    :y="point.y - 10"
                    text-anchor="middle"
                    class="share-label-txt"
                  >
                    {{ point.shareLabel }}
                  </text>
                </g>
              </g>

              <text
                v-for="(mes, index) in chartMetrics?.meses"
                :key="mes"
                :x="(chartMetrics?.startX || 84) + ((chartMetrics?.gap || 0) * index)"
                y="314"
                text-anchor="middle"
                class="svg-txt"
              >
                {{ mes }}
              </text>
            </svg>
          </div>

          <aside class="sidebar">
            <div class="side-title">% Share Mercado</div>
            <div class="side-subtitle">Base: {{ chartMetrics?.latestShareMonth }}</div>
            <div v-for="marca in chartMetrics?.sidebarShares" :key="marca.nome" class="side-item">
              <img :src="`/logos/lubrificantes/${marcasConfig[marca.nome]?.logo || 'generic'}.png`" class="brand-logo-img" />
              <span class="name" :class="{ 'bold-txt': state.empresaSelecionada === marca.nome }">{{ marca.nome }}</span>
              <span class="badge" :class="{ orange: state.empresaSelecionada === marca.nome }">{{ formatPercent(marca.share) }}</span>
            </div>
          </aside>
        </div>
      </section>

      <footer class="ia-legend">
        <div class="ia-ico">✦</div>
        <div class="ia-content">
          <div class="ia-title">LubAssist IA <span class="ia-tag">Análise Ativa</span></div>
          <p>
            Exibindo <b>{{ selectedEmpresaLabel }}</b> com
            <b>{{ state.segmentos.join(' + ') }}</b>
            no período de <b>{{ formatDateDisplay(state.dataDe) }}</b> a <b>{{ formatDateDisplay(state.dataAte) }}</b>.
          </p>
        </div>
      </footer>
    </div>
  </div>
</template>

<style scoped>
.pbi-layout {
  background: #f3f3f3;
  min-height: 100vh;
  padding: 15px;
  font-family: 'Segoe UI', sans-serif;
}

.pbi-content {
  display: flex;
  flex-direction: column;
  gap: 15px;
  min-width: 1200px;
  transition: opacity 0.3s;
}

.loading-opacity {
  opacity: 0.5;
  pointer-events: none;
}

.pbi-filters {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
  background: white;
  padding: 10px 20px;
  border-radius: 8px;
  margin-bottom: 10px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.filter-left {
  display: flex;
  align-items: flex-end;
  gap: 16px;
  flex-wrap: wrap;
}

.filter-block {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.filter-block label {
  font-size: 11px;
  font-weight: 700;
  color: #6b7280;
  text-transform: uppercase;
}

.select-control {
  min-width: 220px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: white;
  padding: 9px 12px;
  font-size: 12px;
  font-weight: 600;
  color: #111827;
}

.segment-picker {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.segment-option {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  border: 1px solid #d1d5db;
  border-radius: 999px;
  padding: 8px 12px;
  background: #fafafa;
  font-size: 12px;
  font-weight: 600;
  color: #374151;
}

.period-picker {
  display: flex;
  gap: 10px;
  align-items: center;
  background: #f8f9fa;
  padding: 6px 10px;
  border-radius: 8px;
  border: 1px solid #ddd;
}

.date-field {
  display: flex;
  align-items: center;
  gap: 6px;
}

.date-field span {
  font-size: 10px;
  font-weight: 700;
  color: #888;
  text-transform: uppercase;
}

.date-field input {
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 5px 8px;
  font-size: 11px;
  outline: none;
  background: white;
}

.date-field input:focus,
.select-control:focus {
  border-color: #f58220;
}

.btn-clear {
  background: white;
  border: 1px solid #333;
  padding: 8px 15px;
  border-radius: 6px;
  font-weight: 700;
  cursor: pointer;
  font-size: 11px;
}

.top-section {
  display: grid;
  grid-template-columns: 3fr 1.5fr 1fr;
  gap: 15px;
  height: 260px;
}

.kpi-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
}

.card {
  background: white;
  border-radius: 10px;
  border: 1px solid #e0e0e0;
  padding: 12px;
  display: flex;
  flex-direction: column;
}

.kpi-label,
.card-title {
  font-size: 12px;
  font-weight: 700;
  color: #6b7280;
}

.kpi-val {
  font-size: 18px;
  font-weight: 800;
  margin: 8px 0;
}

.kpi-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
  margin-top: auto;
}

.sub {
  font-size: 11px;
  color: #6b7280;
}

.pill {
  background: #e6f7ed;
  color: #008a3e;
  font-size: 10px;
  font-weight: 700;
  padding: 2px 8px;
  border-radius: 4px;
}

.pill.neg {
  background: #fee2e2;
  color: #dc2626;
}

.bars-container,
.segment-container {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  flex: 1;
  gap: 8px;
  margin-top: 10px;
}

.bar-col,
.seg-col {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-end;
  height: 100%;
  flex: 1;
}

.bar {
  width: 50%;
  background: #cfd4d9;
  border-radius: 3px 3px 0 0;
  min-height: 2px;
}

.bar.orange {
  background: #f58220;
}

.seg-bar {
  width: 35px;
  border-radius: 3px 3px 0 0;
}

.bar-txt {
  font-size: 9px;
  font-weight: 700;
  margin-bottom: 6px;
  text-align: center;
}

.bar-label {
  font-size: 10px;
  color: #999;
  margin-top: 5px;
}

.bold {
  font-weight: 700;
}

.main-viz {
  min-height: 500px;
}

.viz-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.viz-info h3 {
  margin: 0;
  font-size: 18px;
}

.viz-info p {
  margin: 4px 0 0;
  color: #6b7280;
  font-size: 12px;
}

.caption {
  display: inline-block;
  margin-top: 6px;
  color: #9ca3af;
  font-size: 11px;
  font-weight: 700;
}

.viz-legend {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin: 15px 0;
  padding-bottom: 10px;
  border-bottom: 1px solid #f0f0f0;
}

.leg-item {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 10px;
  border: 1px solid #e5e7eb;
  border-radius: 999px;
  background: white;
  cursor: pointer;
  font-size: 11px;
  color: #4b5563;
}

.leg-item.hidden {
  opacity: 0.45;
}

.leg-item.active {
  border-color: #f58220;
  color: #f58220;
  font-weight: 700;
}

.dot {
  width: 12px;
  height: 3px;
  border-radius: 2px;
}

.viz-body {
  display: flex;
  gap: 20px;
  flex: 1;
}

.svg-area {
  flex: 1;
  padding-left: 10px;
}

.sidebar {
  width: 240px;
  border-left: 1px solid #eee;
  padding-left: 15px;
}

.side-title {
  font-size: 13px;
  font-weight: 800;
  color: #111827;
}

.side-subtitle {
  margin: 4px 0 12px;
  color: #9ca3af;
  font-size: 11px;
}

.side-item {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 8px;
}

.brand-logo-img {
  width: 24px;
  height: 24px;
  object-fit: contain;
}

.name {
  flex: 1;
  font-size: 11px;
  color: #4b5563;
}

.bold-txt {
  font-weight: 800;
  color: #f58220;
}

.badge {
  font-size: 10px;
  background: #eee;
  padding: 2px 6px;
  border-radius: 4px;
  min-width: 52px;
  text-align: center;
}

.badge.orange {
  background: #f58220;
  color: white;
}

.ia-legend {
  background: #1a1d21;
  color: white;
  padding: 15px;
  border-radius: 10px;
  display: flex;
  gap: 15px;
  align-items: center;
  margin-top: 10px;
}

.ia-ico {
  color: #f58220;
  font-size: 20px;
}

.ia-title {
  font-weight: 800;
}

.ia-tag {
  background: #3d3429;
  color: #f58220;
  font-size: 9px;
  padding: 2px 8px;
  border-radius: 4px;
  margin-left: 8px;
}

.svg-txt {
  font-size: 11px;
  fill: #9ca3af;
}

.svg-txt-axis {
  font-size: 10px;
  fill: #9ca3af;
  font-weight: 700;
}

.share-label-txt {
  font-size: 9px;
  fill: #1f2937;
  font-weight: 800;
  pointer-events: none;
}

rect,
.bar,
.seg-bar {
  transition: all 0.3s ease;
}
</style>
