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

// --- COMPUTEDS DE KPIs E FILTROS ---
const availableEmpresas = computed(() => {
  const list = dashboardData.value?.filtros?.empresas || [];
  return list.filter(e => e !== 'IPIRANGA' && e !== 'CHEVRON');
});
const availableSegmentos = computed(() => dashboardData.value?.filtros?.segmentos || []);
const selectedEmpresaLabel = computed(() => (
  state.empresaSelecionada === ALL_BRANDS_VALUE ? 'Todas as marcas' : state.empresaSelecionada
));

const isAnoVigente = computed(() => dashboardData.value?.cardsCalculados?.projecao?.habilitada || false);
const kpiMercado = computed(() => dashboardData.value?.cardsCalculados?.periodoSelecionado?.mercado);
const kpiEmpresa = computed(() => dashboardData.value?.cardsCalculados?.periodoSelecionado?.empresa);
const kpiProjecao = computed(() => dashboardData.value?.cardsCalculados?.projecao);

// --- GRÁFICO ANUAL ---
const annualChartData = computed(() => {
  const comp = dashboardData.value?.graficoAnualComparativo;
  if (!comp?.barrasMercadoTotal) return null;
  const maxVal = Math.max(...comp.barrasMercadoTotal.map(i => i.valor), 1);
  return comp.barrasMercadoTotal.map(item => {
    const valorMarca = comp.linhaMarcaSelecionada?.find(l => l.ano === item.ano)?.valor || 0;
    return {
      ano: item.ano,
      valorTotal: item.valor,
      valorMarca: valorMarca,
      percentualMarca: (valorMarca / maxVal) * 100,
      percentualTotal: (item.valor / maxVal) * 100
    };
  });
});

// --- GRÁFICO MENSAL (AJUSTES 1 E 2 APLICADOS AQUI) ---
const chartMetrics = computed(() => {
  const mensal = dashboardData.value?.graficoMensalComparativo;
  const original = dashboardData.value?.graficos?.vendasPorMes;
  if (!mensal || !original) return null;

  const meses = original.meses || [];
  const marketVolumes = mensal.barrasMercadoTotal.map(v => Number(v.valor || 0));
  
  // AJUSTE 2: Encontrar o último índice que possui dados (volume > 0)
  let lastValidIndex = -1;
  for (let i = marketVolumes.length - 1; i >= 0; i--) {
    if (marketVolumes[i] > 0) {
      lastValidIndex = i;
      break;
    }
  }

  const startX = 84;
  const chartWidth = 850;
  const chartBottom = 286;
  const chartHeight = 230;
  const gap = meses.length > 1 ? chartWidth / (meses.length - 1) : 0;
  const maxMarketVolume = Math.max(...marketVolumes, 1);

  // AJUSTE 1: Barras saindo da base (0%)
  const bars = marketVolumes.map((volume, index) => {
    const height = (volume / maxMarketVolume) * 230; // Usando altura total disponível
    return {
      x: startX + (gap * index) - 18,
      y: chartBottom - height, // Posição calculada para "subir" da base
      height,
      width: 36,
      value: volume
    };
  });

  // AJUSTE 2: Linhas truncadas no último mês com dados
  const allLines = mensal.linhasShareMarcas
    .filter(s => !state.hiddenEmpresas.includes(s.empresa))
    .map((serie) => {
      const isHighlighted = state.empresaSelecionada === serie.empresa;
      // Filtramos os valores para que a linha não continue após o lastValidIndex
      const validValores = serie.valores.slice(0, lastValidIndex + 1);
      
      return {
        empresa: serie.empresa,
        cor: marcasConfig[serie.empresa]?.cor || '#8c8c8c',
        isHighlighted,
        strokeWidth: isHighlighted ? 3.6 : 1.8,
        strokeOpacity: isHighlighted ? 1 : 0.45,
        points: validValores.map((val, idx) => ({
          x: startX + (gap * idx),
          y: getShareY(val, chartBottom, chartHeight),
          shareLabel: formatPercent(val)
        })),
      };
    });

  const sidebarShares = (dashboardData.value?.marcas || [])
    .filter(m => m.nome !== 'IPIRANGA' && m.nome !== 'CHEVRON')
    .sort((a, b) => b.share - a.share);

  return { meses, bars, allLines, sidebarShares, startX, gap, chartBottom, lastValidIndex,
    yTicks: SHARE_TICKS.map(tick => ({ value: tick, y: getShareY(tick, chartBottom, chartHeight) }))
  };
});

// --- FORMATAÇÕES E AUXILIARES ---
function getShareY(share, chartBottom = 286, chartHeight = 230) {
  const safeShare = Math.max(0, Math.min(100, Number(share || 0)));
  const adjustedShare = safeShare * 0.6; // Mantém as linhas na parte de baixo
  const levels = [0, ...SHARE_TICKS];
  const totalSegments = levels.length - 1;
  for (let i = 0; i < totalSegments; i++) {
    const start = levels[i]; const end = levels[i+1];
    if (adjustedShare <= end || i === totalSegments - 1) {
      const fraction = end > start ? (adjustedShare - start) / (end - start) : 0;
      const position = i + Math.max(0, Math.min(1, fraction));
      return chartBottom - ((position / totalSegments) * chartHeight);
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
function formatDateDisplay(d) { if (!d) return ''; const [y, m] = d.split('-'); return `${m}/${y}`; }

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
    if (json.status === 'ok') {
      if (!state.dataDe) initializeFiltersFromResponse(json.data);
      dashboardData.value = json.data;
    }
  } catch (e) { console.error(e); } finally { loading.value = false; }
}

function initializeFiltersFromResponse(data) {
  const f = data?.filtros || {};
  if (f.ultimaCompetencia) {
    const ano = parseInt(f.ultimaCompetencia.ano);
    const mes = parseInt(f.ultimaCompetencia.mes);
    state.dataAte = `${ano}-${String(mes).padStart(2, '0')}-01`;
    state.dataDe = `${ano - 1}-${String(mes).padStart(2, '0')}-01`;
  }
}

function updateDateFromMonthInput(type, val) {
  if (!val) return;
  const [y, m] = val.split('-');
  const dateStr = `${y}-${m}-01`;
  if (type === 'de') state.dataDe = dateStr; else state.dataAte = dateStr;
}

const monthValueDe = computed(() => state.dataDe?.substring(0, 7) || '');
const monthValueAte = computed(() => state.dataAte?.substring(0, 7) || '');

function toggleSegmento(id) {
  state.segmentos = state.segmentos.includes(id) 
    ? (state.segmentos.length > 1 ? state.segmentos.filter(i => i !== id) : state.segmentos)
    : [...state.segmentos, id];
}

function toggleLineVisibility(empresa) {
  state.hiddenEmpresas = state.hiddenEmpresas.includes(empresa)
    ? state.hiddenEmpresas.filter(i => i !== empresa) : [...state.hiddenEmpresas, empresa];
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
            <div class="date-field"><span>De</span><input type="month" :value="monthValueDe" @input="updateDateFromMonthInput('de', $event.target.value)" /></div>
            <div class="date-field"><span>Até</span><input type="month" :value="monthValueAte" @input="updateDateFromMonthInput('ate', $event.target.value)" /></div>
          </div>
        </div>
      </div>
      <button class="btn-clear" @click="state.empresaSelecionada = ALL_BRANDS_VALUE; fetchDashboard()">Limpar filtros</button>
    </header>

    <div class="pbi-content" :class="{ 'loading-opacity': loading }">
      <section class="top-section">
        <div class="kpi-grid">
          <div class="card kpi-card" v-if="kpiMercado">
            <div class="kpi-label">MERCADO: {{ kpiMercado.titulo }}</div>
            <div class="kpi-val">{{ formatVolume(kpiMercado.valor) }}</div>
            <div class="kpi-footer">
              <span class="sub">{{ formatVolume(kpiMercado.comparativo) }}</span>
              <span class="pill" :class="{ neg: kpiMercado.variacaoPercentual < 0 }">{{ formatPercent(kpiMercado.variacaoPercentual) }} YoY</span>
            </div>
          </div>
          <div class="card kpi-card" style="background: #fff6ed;" v-if="kpiEmpresa">
            <div class="kpi-label">{{ state.empresaSelecionada }}: {{ kpiEmpresa.titulo }}</div>
            <div class="kpi-val">{{ formatVolume(kpiEmpresa.valor) }}</div>
            <div class="kpi-footer">
              <span class="sub">{{ formatVolume(kpiEmpresa.comparativo) }}</span>
              <span class="pill" :class="{ neg: kpiEmpresa.variacaoPercentual < 0 }">{{ formatPercent(kpiEmpresa.variacaoPercentual) }} YoY</span>
            </div>
          </div>
          <div v-if="isAnoVigente && kpiProjecao" class="card kpi-card" style="background: #f0f7ff;">
            <div class="kpi-label">Análise Mercado Projeção (Ano)</div>
            <div class="kpi-val">{{ formatVolume(kpiProjecao.valor) }}</div>
            <div class="kpi-footer">
              <span class="sub">Vs Ano Anterior Total</span>
              <span class="pill" :class="{ neg: kpiProjecao.variacaoPercentual < 0 }">{{ formatPercent(kpiProjecao.variacaoPercentual) }}</span>
            </div>
          </div>
        </div>

        <div class="card chart-annual-expanded">
          <div class="card-title">Vendas Anual (mil m³): Mercado vs {{ selectedEmpresaLabel }}</div>
          <div class="annual-comparison-container">
            <div class="annual-bars-container">
              <div v-for="item in annualChartData" :key="item.ano" class="annual-bar-col">
                <div class="bar-wrapper">
                  <span class="bar-txt">{{ formatVolumeAnualMil(item.valorTotal) }}</span>
                  <div class="bar-total" :style="{ height: `${item.percentualTotal * 0.8}%` }">
                     <div v-if="state.empresaSelecionada !== ALL_BRANDS_VALUE && item.valorMarca > 0" 
                       class="bar-marca-inner" 
                       :style="{ height: `${(item.valorMarca / item.valorTotal) * 100}%`, backgroundColor: marcasConfig[state.empresaSelecionada]?.cor || '#f58220' }">
                    </div>
                  </div>
                </div>
                <span class="bar-label">{{ item.ano }}</span>
              </div>
            </div>
            <div class="annual-legend">
              <div class="legend-item"><div class="legend-color total"></div><span>Total Mercado</span></div>
              <div v-if="state.empresaSelecionada !== ALL_BRANDS_VALUE" class="legend-item">
                <div class="legend-color marca" :style="{ backgroundColor: marcasConfig[state.empresaSelecionada]?.cor || '#f58220' }"></div>
                <span>{{ state.empresaSelecionada }}</span>
              </div>
            </div>
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
            <p class="viz-note">* Barras cinza = Volume mercado (Base 0%) | Linhas = % Share (Truncado no último mês)</p>
          </div>
        </div>

        <div class="viz-legend">
          <button v-for="empresa in availableEmpresas" :key="empresa" class="leg-item" 
            :class="{ hidden: state.hiddenEmpresas.includes(empresa), active: state.empresaSelecionada === empresa }" @click="toggleLineVisibility(empresa)">
            <span class="dot" :style="{ background: marcasConfig[empresa]?.cor || '#8c8c8c' }"></span>
            <span>{{ empresa }}</span>
          </button>
        </div>

        <div class="viz-body">
          <div class="svg-area">
            <svg viewBox="0 0 1000 380" preserveAspectRatio="none">
              <!-- AJUSTE 1: Barras saindo do eixo zero (chartBottom) -->
              <g v-for="(bar, index) in chartMetrics?.bars" :key="index">
                <rect v-if="bar.value > 0" :x="bar.x" :y="bar.y" :width="bar.width" :height="bar.height" fill="#cbd5e1" rx="4" opacity="0.35" />
                <text v-if="bar.value > 0" :x="bar.x + bar.width/2" :y="bar.y - 10" text-anchor="middle" class="svg-txt-axis" style="fill: #475569">{{ formatVolumeAnualMil(bar.value) }}k</text>
              </g>

              <!-- Grade Y (Share %) -->
              <g v-for="tick in chartMetrics?.yTicks" :key="tick.value">
                <text x="44" :y="tick.y + 4" text-anchor="end" class="svg-txt-axis">{{ tick.value }}%</text>
                <line x1="56" :y1="tick.y" x2="950" :y2="tick.y" stroke="#e2e8f0" stroke-dasharray="4" />
              </g>

              <!-- AJUSTE 2: Linhas truncadas (não descem para zero após os dados) -->
              <g v-for="line in chartMetrics?.allLines" :key="line.empresa">
                <polyline :points="line.points.map(p => `${p.x},${p.y}`).join(' ')" fill="none" :stroke="line.cor" :stroke-width="line.strokeWidth" :stroke-opacity="line.strokeOpacity" />
                <circle v-for="p in line.points" :key="p.mes" :cx="p.x" :cy="p.y" :r="line.isHighlighted ? 5 : 3.5" :fill="line.cor" stroke="white" stroke-width="1.5" />
              </g>

              <text v-for="(mes, i) in chartMetrics?.meses" :key="mes" :x="(chartMetrics?.startX || 84) + ((chartMetrics?.gap || 0) * i)" y="360" text-anchor="middle" class="svg-txt">{{ mes }}</text>
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

    <footer class="ia-legend">
      <div class="ia-ico">✦</div>
      <div class="ia-content">
        <div class="ia-title">LubAssist IA <span class="ia-tag">Análise Ativa</span></div>
        <p>Exibindo <b>{{ selectedEmpresaLabel }}</b> com <b>{{ state.segmentos.join(' + ') }}</b> no período de <b>{{ formatDateDisplay(state.dataDe) }}</b> a <b>{{ formatDateDisplay(state.dataAte) }}</b>.</p>
      </div>
    </footer>
  </div>
</template>

<style scoped>
/* ESTILOS PRESERVADOS INTEGRALMENTE */
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
.date-field input { border: 1px solid #ccc; border-radius: 4px; padding: 6px; }
.btn-clear { background: white; border: 1px solid #333; padding: 8px 15px; border-radius: 6px; font-weight: 700; cursor: pointer; font-size: 11px; }

.top-section { display: grid; grid-template-columns: 1.8fr 4.2fr 1fr; gap: 15px; min-height: 320px; }

.kpi-grid { display: grid; grid-template-columns: 1fr; gap: 10px; }
.card { background: white; border-radius: 10px; border: 1px solid #e0e0e0; padding: 12px; display: flex; flex-direction: column; }
.kpi-label, .card-title { font-size: 12px; font-weight: 700; color: #6b7280; }
.kpi-val { font-size: 18px; font-weight: 800; margin: 8px 0; }
.kpi-footer { display: flex; justify-content: space-between; margin-top: auto; }
.sub { font-size: 11px; color: #6b7280; }
.pill { background: #e6f7ed; color: #008a3e; font-size: 10px; font-weight: 700; padding: 2px 8px; border-radius: 4px; }
.pill.neg { background: #fee2e2; color: #dc2626; }

.chart-annual-expanded { min-height: 280px; flex: 1; }
.annual-comparison-container { position: relative; flex: 1; display: flex; flex-direction: column; }
.annual-bars-container { display: flex; align-items: flex-end; justify-content: space-around; flex: 1; gap: 20px; margin-top: 10px; height: 180px; }
.annual-bar-col { display: flex; flex-direction: column; align-items: center; flex: 1; height: 100%; justify-content: flex-end; }
.bar-wrapper { position: relative; width: 60px; height: 160px; display: flex; align-items: flex-end; justify-content: center; }
.bar-total { width: 40px; background: #cbd5e1; border-radius: 4px 4px 0 0; position: relative; overflow: hidden; }
.bar-marca-inner { width: 100%; position: absolute; bottom: 0; left: 0; transition: height 0.3s ease; }
.bar-txt { font-size: 10px; font-weight: 700; margin-bottom: 4px; text-align: center; }
.bar-label { font-size: 11px; color: #6b7280; margin-top: 8px; font-weight: 600; }

.segment-container { display: flex; align-items: flex-end; justify-content: space-between; flex: 1; gap: 8px; margin-top: 10px; }
.seg-col { display: flex; flex-direction: column; align-items: center; flex: 1; height: 100%; justify-content: flex-end; }
.seg-bar { width: 30px; border-radius: 3px 3px 0 0; }

.main-viz { min-height: 520px; margin-top: 15px; }
.viz-legend { display: flex; flex-wrap: wrap; gap: 10px; margin: 15px 0; padding-bottom: 10px; border-bottom: 1px solid #f0f0f0; }
.leg-item { display: inline-flex; align-items: center; gap: 6px; padding: 4px 12px; border: 1px solid #e5e7eb; border-radius: 20px; background: white; cursor: pointer; font-size: 11px; }
.leg-item.active { border-color: #f58220; color: #f58220; font-weight: 700; }
.dot { width: 12px; height: 3px; border-radius: 2px; }
.viz-body { display: flex; gap: 20px; flex: 1; }
.svg-area { flex: 1; min-height: 400px; }
.sidebar { width: 240px; border-left: 1px solid #eee; padding-left: 15px; }
.side-item { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; }
.brand-logo-img { width: 20px; height: 20px; object-fit: contain; }
.badge.orange { background: #f58220; color: white; }
.svg-txt { font-size: 11px; fill: #9ca3af; }
.svg-txt-axis { font-size: 10px; fill: #9ca3af; font-weight: 700; }

/* Localize este bloco e altere as linhas de background e cores */
.ia-legend { 
  margin-top: 20px; 
  padding: 16px 20px; 
  background: #1e293b; /* Altera para fundo Dark (Azul petróleo escuro) */
  border-radius: 12px; 
  border-left: 4px solid #f58220; 
  display: flex; 
  align-items: flex-start; 
  gap: 12px; 
}

.ia-title { 
  font-weight: 700; 
  font-size: 14px; 
  color: #ffffff; /* Altera o título para Branco */
  margin-bottom: 6px; 
  display: flex; 
  align-items: center; 
  gap: 8px; 
}

.ia-content p { 
  margin: 0; 
  font-size: 12px; 
  color: #e2e8f0; /* Altera o texto descritivo para um Branco acinzentado */
  line-height: 1.4; 
}

.ia-content b { 
  color: #f58220; /* Mantém os destaques em Laranja para boa leitura no dark */
  font-weight: 600; 
}

.ia-ico { font-size: 24px; color: #f58220; }

.ia-tag { background: #f58220; color: white; font-size: 10px; font-weight: 600; padding: 2px 8px; border-radius: 12px; }
</style>
