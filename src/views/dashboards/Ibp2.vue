<script setup>
import { computed, onMounted, reactive, ref, watch } from 'vue';

const API_BASE = 'https://lubx-api.lubconsulta.com.br/bi/ipb';
const DEFAULT_SEGMENT = 'REVENDEDOR';
const ALL_BRANDS_VALUE = 'TODAS';
const SHARE_TICKS = [1, 2, 3, 4, 5, 10, 15, 20, 25, 30, 40, 50, 60, 70, 80, 90, 100, 150];

const loading = ref(true);
const dashboardData = ref(null);

const state = reactive({
  empresaSelecionada: ALL_BRANDS_VALUE,
  segmentos: [DEFAULT_SEGMENT],
  dataDe: '2025-01-01',
  dataAte: '2026-03-01',
  hiddenEmpresas: [],
});

// Estado do Tooltip
const tooltip = reactive({
  show: false,
  x: 0,
  y: 0,
  empresa: '',
  mes: '',
  volume: 0,
  share: 0,
  cor: ''
});

const monthValueDe = computed(() => state.dataDe?.substring(0, 7) || '');
const monthValueAte = computed(() => state.dataAte?.substring(0, 7) || '');

// Função para atualizar o estado adicionando o dia 01 novamente
function updateDateFromMonthInput(type, val) {
  if (!val) return;
  const dateStr = `${val}-01`; // Transforma "2025-07" em "2025-07-01"
  if (type === 'de') state.dataDe = dateStr; 
  else state.dataAte = dateStr;
}

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

// --- COMPUTEDS DE KPIs (Grade 2x2) ---
const kpiMercadoPeriodo = computed(() => dashboardData.value?.cardsCalculados?.periodoSelecionado?.mercado);
const kpiEmpresaPeriodo = computed(() => dashboardData.value?.cardsCalculados?.periodoSelecionado?.empresa);
const projecaoDados = computed(() => dashboardData.value?.cardsCalculados?.projecao);

const kpiMercadoProjecao = computed(() => {
  const p = projecaoDados.value?.mercado;
  if (!p) return null;
  return {
    titulo: 'Análise Mercado - Projeção',
    valor: p.valorProjetado,
    comparativo: p.comparativoAnoAnteriorProjetado,
    variacao: p.comparativoAnoAnteriorProjetado > 0 ? ((p.valorProjetado / p.comparativoAnoAnteriorProjetado) - 1) * 100 : 0
  };
});

const kpiEmpresaProjecao = computed(() => {
  const p = projecaoDados.value?.empresa;
  if (!p) return null;
  return {
    titulo: `Análise ${selectedEmpresaLabel.value} - Projeção`,
    valor: p.valorProjetado,
    comparativo: p.comparativoAnoAnteriorProjetado,
    variacao: p.comparativoAnoAnteriorProjetado > 0 ? ((p.valorProjetado / p.comparativoAnoAnteriorProjetado) - 1) * 100 : 0
  };
});

const availableEmpresas = computed(() => {
  const list = dashboardData.value?.filtros?.empresas || [];
  return list.filter(e => e !== 'IPIRANGA' && e !== 'CHEVRON');
});
const availableSegmentos = computed(() => dashboardData.value?.filtros?.segmentos || []);
const selectedEmpresaLabel = computed(() => (
  state.empresaSelecionada === ALL_BRANDS_VALUE ? 'Todas as marcas' : state.empresaSelecionada
));

// --- GRÁFICO ANUAL (Linha conectada ao topo da barra) ---
const annualChartMetrics = computed(() => {
  const comp = dashboardData.value?.graficoAnualComparativo;
  if (!comp?.barrasMercadoTotal) return null;

  const chartWidth = 600;
  const chartHeight = 220;
  const paddingBottom = 45;
  const paddingTop = 30;
  const usableHeight = chartHeight - paddingTop - paddingBottom;
  const baseLineY = chartHeight - paddingBottom;
  
  const maxVal = Math.max(...comp.barrasMercadoTotal.map(i => i.valor), 1);
  const gap = chartWidth / comp.barrasMercadoTotal.length;
  const barWidth = 35;

  const data = comp.barrasMercadoTotal.map((item, idx) => {
    const valorMarca = comp.linhaMarcaSelecionada?.find(l => l.ano === item.ano)?.valor || 0;
    const sharePercent = item.valor > 0 ? (valorMarca / item.valor) * 100 : 0;
    
    const x = (idx * gap) + (gap / 2);
    const barTotalHeight = (item.valor / maxVal) * usableHeight;
    const barMarcaHeight = (valorMarca / maxVal) * usableHeight;

    return {
      ano: item.ano,
      valorTotal: item.valor,
      valorMarca: valorMarca,
      share: sharePercent,
      x,
      barTotalY: baseLineY - barTotalHeight,
      barTotalHeight,
      barMarcaY: baseLineY - barMarcaHeight,
      barMarcaHeight
    };
  });

  return { data, barWidth, chartWidth, chartHeight, baseLineY };
});

// --- GRÁFICO MENSAL (Com volumes para o Tooltip) ---
const chartMetrics = computed(() => {
  const mensal = dashboardData.value?.graficoMensalComparativo;
  const original = dashboardData.value?.graficos?.vendasPorMes;
  if (!mensal || !original) return null;

  const meses = original.meses || [];
  const marketVolumes = mensal.barrasMercadoTotal.map(v => Number(v.valor || 0));
  
  let lastValidIndex = -1;
  for (let i = marketVolumes.length - 1; i >= 0; i--) {
    if (marketVolumes[i] > 0) { lastValidIndex = i; break; }
  }

  const baseStartX = 84;
  const chartWidth = 850;
  const chartBottom = 400;
  const chartHeight = 340;
  
  let gap = meses.length > 1 ? chartWidth / (meses.length - 1) : 0;
  let startX = baseStartX;
  if (gap > 150 && meses.length > 1) {
    gap = 150;
    startX = baseStartX + (chartWidth - (gap * (meses.length - 1))) / 2;
  }

  const bars = marketVolumes.map((volume, index) => ({
    x: startX + (gap * index) - 18,
    y: getShareY(volume / 500, chartBottom, chartHeight),
    height: chartBottom - getShareY(volume / 500, chartBottom, chartHeight),
    width: 36,
    value: volume
  }));

  const allLines = mensal.linhasShareMarcas
    .filter(s => !state.hiddenEmpresas.includes(s.empresa))
    .filter(serie => serie.valores.some(val => val > 0)) 
    .map((serie) => {
      const isHighlighted = state.empresaSelecionada === serie.empresa;
      const volumeSerie = original.seriesVolume?.find(v => v.empresa === serie.empresa);
      return {
        empresa: serie.empresa,
        cor: marcasConfig[serie.empresa]?.cor || '#8c8c8c',
        isHighlighted,
        strokeWidth: isHighlighted ? 3.6 : 1.8,
        strokeOpacity: isHighlighted ? 1 : 0.45,
        points: serie.valores.slice(0, lastValidIndex + 1).map((val, idx) => ({
          x: startX + (gap * idx),
          y: getShareY(val, chartBottom, chartHeight),
          share: val,
          volume: volumeSerie ? volumeSerie.valores[idx] : 0,
        })),
      };
    });

  return { meses, bars, allLines, startX, gap, 
    sidebarShares: (dashboardData.value?.marcas || []).filter(m => m.nome !== 'IPIRANGA' && m.nome !== 'CHEVRON').sort((a, b) => b.share - a.share),
    yTicks: SHARE_TICKS.map(tick => ({ value: tick, y: getShareY(tick, chartBottom, chartHeight) }))
  };
});

// --- AUXILIARES ---
function getShareY(share, chartBottom, chartHeight) {
  const safeShare = Math.max(0, Math.min(250, Number(share || 0)));
  const levels = [0, ...SHARE_TICKS];
  for (let i = 0; i < levels.length - 1; i++) {
    const start = levels[i], end = levels[i+1];
    if (safeShare <= end || i === levels.length - 2) {
      const fraction = end > start ? (safeShare - start) / (end - start) : 0;
      return chartBottom - (((i + fraction) / (levels.length - 1)) * chartHeight);
    }
  }
  return chartBottom;
}

function formatVolume(v) { return `${Number(v || 0).toLocaleString('pt-BR', { maximumFractionDigits: 0 })} m³`; }
function formatVolumeCompact(v) { 
  const n = Number(v || 0);
  return n >= 1000 ? `${(n/1000).toLocaleString('pt-BR', {maximumFractionDigits:1})} mil m³` : `${n.toLocaleString('pt-BR')} m³`;
}
function formatVolumeAnualMil(v) { return (Number(v || 0) / 1000).toLocaleString('pt-BR', { maximumFractionDigits: 1 }); }
function formatPercent(v) { return `${Number(v || 0).toLocaleString('pt-BR', { minimumFractionDigits: 1, maximumFractionDigits: 1 })}%`; }
function formatDateDisplay(d) { if (!d) return ''; const [y, m, d_] = d.split('-'); return `${m}/${y}`; }

async function fetchDashboard() {
  loading.value = true;
  try {
    const params = new URLSearchParams();
    params.append('empresa', state.empresaSelecionada);
    params.append('dataDe', state.dataDe);
    params.append('dataAte', state.dataAte);
    state.segmentos.forEach(s => params.append('segmento', s));
    const response = await fetch(`${API_BASE}/dashboard?${params.toString()}`);
    const json = await response.json();
    if (json.status === 'ok') dashboardData.value = json.data;
  } catch (e) { console.error(e); } finally { loading.value = false; }
}

function handleMouseMove(event, empresa, point, mes) {
  tooltip.show = true;
  tooltip.x = event.clientX;
  tooltip.y = event.clientY;
  tooltip.empresa = empresa;
  tooltip.mes = mes;
  tooltip.volume = point.volume;
  tooltip.share = point.share;
  tooltip.cor = marcasConfig[empresa]?.cor || '#8c8c8c';
}

function toggleSegmento(id) {
  state.segmentos = state.segmentos.includes(id) ? (state.segmentos.length > 1 ? state.segmentos.filter(i => i !== id) : state.segmentos) : [...state.segmentos, id];
}

function toggleLineVisibility(empresa) {
  state.hiddenEmpresas = state.hiddenEmpresas.includes(empresa) ? state.hiddenEmpresas.filter(i => i !== empresa) : [...state.hiddenEmpresas, empresa];
}

onMounted(fetchDashboard);
watch(() => [state.empresaSelecionada, state.dataDe, state.dataAte, state.segmentos], fetchDashboard);
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
            <div class="date-field">
              <span>De</span>
              <input 
                type="month" 
                :value="monthValueDe" 
                @input="updateDateFromMonthInput('de', $event.target.value)" 
              />
            </div>
            <div class="date-field">
              <span>Até</span>
              <input 
                type="month" 
                :value="monthValueAte" 
                @input="updateDateFromMonthInput('ate', $event.target.value)" 
              />
            </div>
          </div>
        </div>
      </div>
      <button class="btn-clear" @click="state.empresaSelecionada = ALL_BRANDS_VALUE; fetchDashboard()">Limpar filtros</button>
    </header>

    <div class="pbi-content" :class="{ 'loading-opacity': loading }">
      <section class="top-section">
        <div class="kpi-grid">
          <div class="card kpi-card" v-if="kpiMercadoPeriodo">
            <div class="kpi-label">MERCADO: {{ kpiMercadoPeriodo.titulo }}</div>
            <div class="kpi-val">{{ formatVolume(kpiMercadoPeriodo.valor) }}</div>
            <div class="kpi-footer"><span class="sub">{{ formatVolume(kpiMercadoPeriodo.comparativo) }}</span><span class="pill" :class="{ neg: kpiMercadoPeriodo.variacaoPercentual < 0 }">{{ formatPercent(kpiMercadoPeriodo.variacaoPercentual) }} YoY</span></div>
          </div>
          <div class="card kpi-card" v-if="kpiMercadoProjecao">
            <div class="kpi-label">{{ kpiMercadoProjecao.titulo }}</div>
            <div class="kpi-val">{{ formatVolume(kpiMercadoProjecao.valor) }}</div>
            <div class="kpi-footer"><span class="sub">{{ formatVolume(kpiMercadoProjecao.comparativo) }}</span><span class="pill" :class="{ neg: kpiMercadoProjecao.variacaoPercentual < 0 }">{{ formatPercent(kpiMercadoProjecao.variacaoPercentual) }} YoY</span></div>
          </div>
          <div class="card kpi-card brand-bg" v-if="kpiEmpresaPeriodo">
            <div class="kpi-label">{{ selectedEmpresaLabel }}: {{ kpiEmpresaPeriodo.titulo }}</div>
            <div class="kpi-val">{{ formatVolume(kpiEmpresaPeriodo.valor) }}</div>
            <div class="kpi-footer"><span class="sub">{{ formatVolume(kpiEmpresaPeriodo.comparativo) }}</span><span class="pill" :class="{ neg: kpiEmpresaPeriodo.variacaoPercentual < 0 }">{{ formatPercent(kpiEmpresaPeriodo.variacaoPercentual) }} YoY</span></div>
          </div>
          <div class="card kpi-card brand-bg" v-if="kpiEmpresaProjecao">
            <div class="kpi-label">{{ kpiEmpresaProjecao.titulo }}</div>
            <div class="kpi-val">{{ formatVolume(kpiEmpresaProjecao.valor) }}</div>
            <div class="kpi-footer"><span class="sub">{{ formatVolume(kpiEmpresaProjecao.comparativo) }}</span><span class="pill" :class="{ neg: kpiEmpresaProjecao.variacaoPercentual < 0 }">{{ formatPercent(kpiEmpresaProjecao.variacaoPercentual) }} YoY</span></div>
          </div>
        </div>

        <!-- GRÁFICO ANUAL -->
        <div class="card chart-annual-expanded">
          <div class="card-title">Vendas Anual (mil m³): Mercado vs {{ selectedEmpresaLabel }}</div>
          <div class="annual-svg-container" v-if="annualChartMetrics">
            <svg viewBox="0 0 600 240" preserveAspectRatio="xMidYMid meet">
              <g v-for="item in annualChartMetrics.data" :key="item.ano">
                <rect :x="item.x - annualChartMetrics.barWidth/2" :y="item.barTotalY" :width="annualChartMetrics.barWidth" :height="item.barTotalHeight" fill="#cbd5e1" rx="2" />
                <rect v-if="state.empresaSelecionada !== ALL_BRANDS_VALUE && item.valorMarca > 0" :x="item.x - annualChartMetrics.barWidth/2" :y="item.barMarcaY" :width="annualChartMetrics.barWidth" :height="item.barMarcaHeight" :fill="marcasConfig[state.empresaSelecionada]?.cor" rx="2" />
                <text :x="item.x" :y="item.barTotalY - 8" text-anchor="middle" class="svg-annual-vol">{{ formatVolumeAnualMil(item.valorTotal) }}</text>
                <text v-if="state.empresaSelecionada !== ALL_BRANDS_VALUE && item.share > 0" :x="item.x" :y="annualChartMetrics.baseLineY + 18" text-anchor="middle" class="svg-annual-share" :fill="marcasConfig[state.empresaSelecionada]?.cor">{{ formatPercent(item.share) }}</text>
                <text :x="item.x" :y="235" text-anchor="middle" class="svg-annual-year">{{ item.ano }}</text>
              </g>
              <polyline v-if="state.empresaSelecionada !== ALL_BRANDS_VALUE" fill="none" :stroke="marcasConfig[state.empresaSelecionada]?.cor" stroke-width="2.5" stroke-linecap="round" :points="annualChartMetrics.data.filter(d => d.valorMarca > 0).map(d => `${d.x},${d.barMarcaY}`).join(' ')" />
              <circle v-if="state.empresaSelecionada !== ALL_BRANDS_VALUE" v-for="item in annualChartMetrics.data.filter(d => d.valorMarca > 0)" :key="'c-'+item.ano" :cx="item.x" :cy="item.barMarcaY" r="4" :fill="marcasConfig[state.empresaSelecionada]?.cor" stroke="white" stroke-width="1.5" />
            </svg>
          </div>
          <div class="annual-legend">
            <div class="legend-item"><div class="legend-color total"></div><span>Mercado</span></div>
            <div v-if="state.empresaSelecionada !== ALL_BRANDS_VALUE" class="legend-item"><div class="legend-color marca" :style="{ background: marcasConfig[state.empresaSelecionada]?.cor }"></div><span>{{ state.empresaSelecionada }}</span></div>
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

      <!-- GRÁFICO MENSAL COM HOVER -->
      <section class="card main-viz">
        <div class="viz-header">
          <div class="viz-info"><h3>Vendas de Lubrificantes por Mês</h3><p class="viz-note">* Barras cinza = Mercado | Linhas = % Share</p></div>
        </div>
        <div class="viz-legend">
          <button v-for="empresa in availableEmpresas" :key="empresa" class="leg-item" :class="{ active: state.empresaSelecionada === empresa, hidden: state.hiddenEmpresas.includes(empresa) }" @click="toggleLineVisibility(empresa)">
            <span class="dot" :style="{ background: marcasConfig[empresa]?.cor || '#8c8c8c' }"></span><span>{{ empresa }}</span>
          </button>
        </div>
        <div class="viz-body">
          <div class="svg-area">
            <svg viewBox="0 0 1000 450" preserveAspectRatio="none">
              <g v-for="(bar, index) in chartMetrics?.bars" :key="index">
                <rect v-if="bar.value > 0" :x="bar.x" :y="bar.y" :width="bar.width" :height="bar.height" fill="#cbd5e1" rx="4" opacity="0.35" />
                <text v-if="bar.value > 0" :x="bar.x + bar.width/2" :y="bar.y - 10" text-anchor="middle" class="svg-txt-axis">{{ formatVolumeAnualMil(bar.value) }}k</text>
              </g>
              <g v-for="tick in chartMetrics?.yTicks" :key="tick.value">
                <text v-if="tick.value <= 30" x="44" :y="tick.y + 4" text-anchor="end" class="svg-txt-axis">{{ tick.value }}%</text>
                <line v-if="tick.value <= 30 || tick.value === 100" x1="56" :y1="tick.y" x2="950" :y2="tick.y" stroke="#e2e8f0" stroke-dasharray="4" />
              </g>
              <g v-for="line in chartMetrics?.allLines" :key="line.empresa">
                <polyline :points="line.points.map(p => `${p.x},${p.y}`).join(' ')" fill="none" :stroke="line.cor" :stroke-width="line.strokeWidth" :stroke-opacity="line.strokeOpacity" />
                <circle v-for="(p, idx) in line.points" :key="idx" :cx="p.x" :cy="p.y" :r="line.isHighlighted ? 6 : 4.5" :fill="line.cor" stroke="white" stroke-width="1.5" class="chart-point" @mousemove="handleMouseMove($event, line.empresa, p, chartMetrics.meses[idx])" @mouseleave="tooltip.show = false" />
              </g>
              <text v-for="(mes, i) in chartMetrics?.meses" :key="mes" :x="(chartMetrics?.startX || 84) + ((chartMetrics?.gap || 0) * i)" y="430" text-anchor="middle" class="svg-txt">{{ mes }}</text>
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

    <!-- TOOLTIP REATIVO -->
    <div v-if="tooltip.show" class="custom-tooltip" :style="{ top: (tooltip.y - 100) + 'px', left: (tooltip.x + 15) + 'px' }">
      <div class="tooltip-header" :style="{ borderLeftColor: tooltip.cor }">
        <strong>{{ tooltip.empresa }}</strong><span>{{ tooltip.mes }}</span>
      </div>
      <div class="tooltip-body">
        <div class="tooltip-row"><span>Volume:</span> <b>{{ formatVolume(tooltip.volume) }}</b></div>
        <div class="tooltip-row"><span>Share:</span> <b>{{ formatPercent(tooltip.share) }}</b></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.pbi-layout { background: #f3f3f3; min-height: 100vh; padding: 15px; font-family: 'Segoe UI', sans-serif; }
.pbi-content { display: flex; flex-direction: column; gap: 15px; min-width: 1200px; }
.loading-opacity { opacity: 0.5; pointer-events: none; }
.pbi-filters { display: flex; justify-content: space-between; align-items: center; background: white; padding: 10px 20px; border-radius: 8px; margin-bottom: 10px; }
.filter-left { display: flex; align-items: flex-end; gap: 16px; }
.filter-block label { font-size: 11px; font-weight: 700; color: #6b7280; text-transform: uppercase; margin-bottom: 4px; display: block; }
.select-control { min-width: 200px; border: 1px solid #d1d5db; border-radius: 6px; padding: 8px; font-size: 12px; }
.segment-picker { display: flex; gap: 8px; }
.segment-option { display: inline-flex; align-items: center; gap: 4px; border: 1px solid #d1d5db; border-radius: 20px; padding: 4px 12px; font-size: 11px; background: #fafafa; cursor: pointer; }
.period-picker { 
  display: flex; 
  gap: 12px; 
  background: #f8fafc; 
  padding: 6px 12px; 
  border-radius: 8px; 
  border: 1px solid #cbd5e1; 
  align-items: center;
}

.date-field { 
  display: flex; 
  align-items: center; 
  gap: 8px; 
  font-size: 11px; 
  color: #64748b;
  font-weight: 600;
}

.date-field input { 
  border: 1px solid #e2e8f0; 
  border-radius: 6px; 
  padding: 5px 8px; 
  font-size: 12px; 
  color: #1e293b;
  cursor: pointer;
  background: white;
  outline: none;
  transition: border-color 0.2s;
}

.date-field input:hover {
  border-color: #f58220;
}

/* Customização do ícone do calendário para alguns browsers */
.date-field input::-webkit-calendar-picker-indicator {
  cursor: pointer;
  filter: invert(48%) sepia(13%) stroke(1px, #f58220); /* Opcional: muda a cor do ícone */
}
.btn-clear { background: white; border: 1px solid #333; padding: 8px 15px; border-radius: 6px; font-weight: 700; cursor: pointer; font-size: 11px; }

.top-section { display: grid; grid-template-columns: 3.5fr 3fr 1fr; gap: 15px; }
.kpi-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; align-content: start; }
.card { 
  background: white; 
  border-radius: 10px; 
  border: 1px solid #e0e0e0; 
  padding: 22px; /* Aumentei de 15px para 22px para dar mais corpo */
  display: flex; 
  flex-direction: column;
  min-height: 180px; /* Adicione uma altura mínima se quiser que sejam mais altos */
}
.brand-bg { background: #fff6ed !important; }

.kpi-label { font-size: 10px; font-weight: 700; color: #6b7280; text-transform: uppercase; }
.kpi-val { font-size: 20px; font-weight: 800; margin: 8px 0; color: #1e293b; }
.kpi-footer { display: flex; justify-content: space-between; align-items: center; }
.sub { font-size: 10px; color: #94a3b8; }
.pill { background: #e6f7ed; color: #008a3e; font-size: 10px; font-weight: 700; padding: 2px 8px; border-radius: 4px; }
.pill.neg { background: #fee2e2; color: #dc2626; }

.chart-annual-expanded { min-height: 310px; align-self: stretch; }
.annual-svg-container { flex: 1; margin-top: 10px; }
.svg-annual-vol { font-size: 10px; font-weight: 700; fill: #475569; }
.svg-annual-year { font-size: 11px; fill: #64748b; font-weight: 600; }
.svg-annual-share { font-size: 10px; font-weight: 800; }
.annual-legend { display: flex; gap: 15px; margin-top: 10px; justify-content: center; }
.legend-item { display: flex; align-items: center; gap: 5px; font-size: 10px; font-weight: 600; color: #64748b; }
.legend-color { width: 12px; height: 12px; border-radius: 2px; }
.legend-color.total { background: #cbd5e1; }
.legend-color.marca { border-radius: 2px; }

.segment-container { display: flex; align-items: flex-end; justify-content: space-around; flex: 1; margin-top: 10px; }
.seg-col { display: flex; flex-direction: column; align-items: center; height: 100%; justify-content: flex-end; }
.seg-bar { width: 30px; border-radius: 4px 4px 0 0; }
.bar-txt { font-size: 10px; font-weight: 700; margin-bottom: 5px; }
.bar-label { font-size: 11px; margin-top: 5px; color: #64748b; }

.main-viz { min-height: 600px; margin-top: 15px; }
.viz-legend { display: flex; flex-wrap: wrap; gap: 8px; margin: 15px 0; }
.leg-item { display: inline-flex; align-items: center; gap: 6px; padding: 4px 12px; border: 1px solid #e5e7eb; border-radius: 20px; background: white; cursor: pointer; font-size: 11px; }
.leg-item.active { border-color: #f58220; color: #f58220; font-weight: 700; }
.leg-item.hidden { opacity: 0.4; text-decoration: line-through; }
.dot { width: 10px; height: 3px; border-radius: 2px; }
.viz-body { display: flex; gap: 20px; flex: 1; }
.svg-area { flex: 1; }



.sidebar { 
  width: 220px; 
  border-left: 1px solid #eee; 
  padding-left: 15px; 
}

.side-item { 
  display: flex; 
  align-items: center; 
  gap: 10px; 
  margin-bottom: 8px;
  width: 100%; /* Garante que use todo o espaço da sidebar */
}

.brand-logo-img { 
  width: 18px; 
  height: 18px; 
  object-fit: contain; 
}

.name {
  font-size: 12px;
  color: #1e293b;
}

.name.bold-txt {
  font-weight: 700;
}

/* ESTA É A MUDANÇA PRINCIPAL */
.badge { 
  background: #f1f5f9; /* Fundo cinza claro para todas as marcas */
  color: #475569; 
  border-radius: 4px; 
  padding: 2px 6px; 
  font-size: 12px; 
  font-weight: 700;
  margin-left: auto; /* Empurra o badge para a extrema direita */
}

/* Quando a marca estiver selecionada, sobrepõe com a cor laranja */
.badge.orange { 
  background: #f58220; 
  color: white; 
  font-size: 12px;
}


.svg-txt-axis { font-size: 10px; fill: #94a3b8; font-weight: 700; }
.svg-txt { font-size: 11px; fill: #94a3b8; }

/* ESTILOS DO TOOLTIP */
.chart-point { cursor: pointer; transition: r 0.2s ease; }
.chart-point:hover { r: 8 !important; }
.custom-tooltip { position: fixed; z-index: 9999; background: white; border: 1px solid #cbd5e1; border-radius: 8px; padding: 10px; box-shadow: 0 10px 15px -3px rgba(0,0,0,0.1); pointer-events: none; font-size: 12px; min-width: 150px; }
.tooltip-header { border-left: 4px solid; padding-left: 8px; margin-bottom: 5px; display: flex; flex-direction: column; }
.tooltip-header strong { font-size: 13px; color: #1e293b; }
.tooltip-header span { font-size: 11px; color: #64748b; }
.tooltip-row { display: flex; justify-content: space-between; margin-top: 2px; }
.tooltip-row span { color: #64748b; }
.tooltip-row b { color: #1e293b; }
</style>
