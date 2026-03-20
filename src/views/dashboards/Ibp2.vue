<script setup>
import { ref, computed } from 'vue';

// --- DADOS DO GRÁFICO DE LINHAS (LUBRIFICANTES) ---
const marcas = [
  { nome: 'YPF', cor: '#00aaff' },
  { nome: 'VIBRA', cor: '#1a237e' },
  { nome: 'VALVOLINE', cor: '#ff6d00' },
  { nome: 'TOTALENERGIES', cor: '#4a148c' },
  { nome: 'RAIZEN', cor: '#d81b60' },
  { nome: 'PETRONAS', cor: '#7e57c2' },
  { nome: 'MOOVE', cor: '#fbc02d' },
  { nome: 'LWART', cor: '#c62828' },
  { nome: 'ICONIC', cor: '#006064' },
  { nome: 'CASTROL', cor: '#2e7d32' }
];

const mesesLista = ['Jan', 'Fev', 'Mar', 'Abr', 'Mai', 'Jun', 'Jul', 'Ago', 'Set', 'Out', 'Nov', 'Dez'];

// Estados de Filtro
const marcasAtivas = ref(marcas.map(m => m.nome));
const mesInicio = ref(0);
const mesFim = ref(11);

const dadosVendasOriginal = {
  'YPF':           [5, 6, 8, 7, 9, 6, 8, 7, 8, 9, 7, 6],
  'VIBRA':         [22, 20, 24, 23, 25, 22, 24, 25, 26, 26, 22, 18],
  'VALVOLINE':     [3, 2, 4, 3, 4, 3, 4, 3, 4, 4, 3, 2],
  'TOTALENERGIES': [5, 4, 6, 5, 7, 5, 6, 6, 5, 7, 5, 4],
  'RAIZEN':        [18, 16, 17, 16, 19, 15, 20, 19, 21, 18, 17, 15],
  'PETRONAS':      [14, 15, 13, 15, 16, 10, 14, 15, 16, 15, 13, 12],
  'MOOVE':         [25, 20, 22, 24, 23, 20, 26, 25, 24, 28, 22, 19],
  'LWART':         [4, 5, 4, 6, 4, 5, 4, 4, 5, 4, 4, 3],
  'ICONIC':        [35, 30, 38, 36, 40, 35, 42, 40, 41, 45, 38, 32],
  'CASTROL':       [8, 7, 9, 8, 10, 8, 9, 9, 8, 10, 8, 7]
};

// --- LÓGICA DO GRÁFICO SVG ---
const larguraSVG = 1000;
const alturaSVG = 350;
const paddingX = 70;
const paddingY = 40;
const maxY = 50;

const mesesFiltrados = computed(() => mesesLista.slice(mesInicio.value, parseInt(mesFim.value) + 1));

const getX = (indexRelativo) => {
  const qtd = mesesFiltrados.value.length;
  if (qtd <= 1) return larguraSVG / 2;
  return paddingX + (indexRelativo * (larguraSVG - paddingX * 2) / (qtd - 1));
};

const getY = (val) => (alturaSVG - paddingY) - (val / maxY * (alturaSVG - paddingY * 2));

const gerarPontos = (nome) => {
  const valores = dadosVendasOriginal[nome].slice(mesInicio.value, parseInt(mesFim.value) + 1);
  return valores.map((v, i) => `${getX(i)},${getY(v)}`).join(' ');
};

const toggleMarca = (nome) => {
  if (marcasAtivas.value.includes(nome)) {
    marcasAtivas.value = marcasAtivas.value.filter(m => m !== nome);
  } else {
    marcasAtivas.value.push(nome);
  }
};

const KpiIcon = `<svg width="32" height="32" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M3 18H5V12H3V18ZM7 18H9V8H7V18ZM11 18H13V14H11V18ZM15 18H17V10H15V18ZM19 18H21V6H19V18Z" fill="#F58220" fill-opacity="0.3"/><path d="M3 15L8 10L12 14L21 5M21 5H16M21 5V10" stroke="#F58220" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/></svg>`;

const kpis = [
    { label: 'ANÁLISE MERCADO - ANUAL', val: '1.230.987 m³', sub: '1.216.760 m³', perc: '1,17%' },
    { label: 'ANÁLISE MERCADO - MENSAL', val: '1.230.987 m³', sub: '1.216.760 m³', perc: '1,17%' },
    { label: 'ANÁLISE MERCADO - PROJEÇÃO', val: '1.230.987 m³', sub: '1.216.760 m³', perc: '1,17%' },
    { label: 'ANÁLISE TOTALENERGIES - ANUAL', val: '16.617 m³', sub: '15.826 m³', perc: '5,00%' },
    { label: 'ANÁLISE TOTALENERGIES - MENSAL', val: '16.617 m³', sub: '15.826 m³', perc: '5,00%' },
    { label: 'ANÁLISE TOTALENERGIES - PROJEÇÃO', val: '16.617 m³', sub: '15.826 m³', perc: '5,00%' },
];

const vendaAnualData = [
    {y:'2017', v:'21.435', n:21435}, {y:'2018', v:'', n:21435}, {y:'2019', v:'19.275', n:19275}, 
    {y:'2020', v:'16.946', n:16946}, {y:'2021', v:'22.097', n:22097}, {y:'2022', v:'18.245', n:18245}, 
    {y:'2023', v:'15.133', n:15133}, {y:'2024', v:'', n:14000}, {y:'2025', v:'16.617', n:16617}, {y:'2026', v:'2.944', n:4000}
];
</script>

<template>
  <div class="pbi-dashboard">
    <!-- FILTROS -->
    <header class="pbi-header">
      <div class="f-group"><span>SELECIONE A EMPRESA:</span> <select><option>TOTALENERGIES</option></select></div>
      <div class="f-group"><span>ANO:</span> <select><option>2025</option></select></div>
      <div class="f-group"><span>DO MÊS:</span> 
        <select v-model="mesInicio">
            <option v-for="(m, i) in mesesLista" :key="i" :value="i">{{ m }}</option>
        </select>
      </div>
      <div class="f-group"><span>ATÉ O MÊS:</span> 
        <select v-model="mesFim">
            <option v-for="(m, i) in mesesLista" :key="i" :value="i">{{ m }}</option>
        </select>
      </div>
      <div class="f-group">
        <span>SEGMENTO:</span>
        <div class="pbi-tabs"><button>CONSUMIDOR</button><button class="active">REVENDEDOR</button></div>
      </div>
      <button class="btn-pbi-red" @click="marcasAtivas = marcas.map(m => m.nome)">Limpar Filtros</button>
    </header>

    <div class="pbi-content">
      <!-- LINHA SUPERIOR RESTAURADA -->
      <div class="top-row">
        <!-- KPIS DETALHADOS -->
        <div class="kpi-grid">
          <div class="pbi-card kpi-card" v-for="(k, i) in kpis" :key="i">
            <div class="kpi-inner">
              <div class="kpi-icon" v-html="KpiIcon"></div>
              <div class="kpi-text">
                <div class="kpi-label">{{ k.label }}</div>
                <div class="kpi-val">{{ k.val }}</div>
                <div class="kpi-sub">
                  <span>{{ k.sub }}</span>
                  <span class="pbi-dot">●</span>
                  <span class="pbi-perc">{{ k.perc }}</span>
                </div>
              </div>
              <div class="kpi-icon mirror" v-html="KpiIcon"></div>
            </div>
          </div>
        </div>

        <!-- VENDA ANUAL (BARRAS) -->
        <div class="pbi-card side-chart">
          <div class="chart-header">VENDA ANUAL</div>
          <div class="venda-anual-bars">
            <div v-for="(v, i) in vendaAnualData" :key="i" class="va-col">
              <span class="va-txt">{{ v.v }}</span>
              <div class="va-bar" :class="{ 'last-bar': i === 9 }" :style="{ height: (v.n / 25000 * 100) + '%' }"></div>
              <span class="va-year">{{ v.y }}</span>
            </div>
          </div>
        </div>

        <!-- SEGMENTO (BARRAS) -->
        <div class="pbi-card side-chart">
          <div class="chart-header text-center">Vendas de Lubrificantes por Segmento</div>
          <div class="seg-viz">
            <div class="seg-bar-group">
              <span class="va-txt">15.239</span>
              <div class="va-bar" style="height: 120px; width: 45px; background: #084594;"></div>
              <span class="va-year font-bold">REVENDEDOR</span>
            </div>
            <div class="seg-bar-group">
              <span class="va-txt">1.378</span>
              <div class="va-bar" style="height: 20px; width: 45px; background: #41b6c4;"></div>
              <span class="va-year font-bold">CONSUMIDOR</span>
            </div>
          </div>
        </div>
      </div>

      <!-- GRÁFICO INFERIOR (O NOVO INTERATIVO) -->
      <div class="pbi-card main-viz-card">
        <div class="chart-header-row">
          <span class="chart-title">Vendas de Lubrificantes por mês</span>
          <div class="pbi-line-legend">
            <div v-for="marca in marcas" :key="marca.nome" 
                 class="leg-item" 
                 :class="{ 'inactive': !marcasAtivas.includes(marca.nome) }"
                 @click="toggleMarca(marca.nome)">
              <span class="leg-line" :style="{ background: marca.cor }"></span>
              <span class="leg-dot" :style="{ borderColor: marca.cor }"></span>
              <span class="leg-text">{{ marca.nome }}</span>
            </div>
          </div>
        </div>
        
        <div class="main-viz-wrapper">
          <svg class="svg-line-chart" :viewBox="`0 0 ${larguraSVG} ${alturaSVG}`" preserveAspectRatio="none">
            <line v-for="n in 6" :key="n" 
              :x1="paddingX" :y1="getY((n-1)*10)" :x2="larguraSVG - paddingX" :y2="getY((n-1)*10)" 
              stroke="#eee" stroke-width="1" />
            
            <text v-for="n in 6" :key="'ty'+n" :x="paddingX - 10" :y="getY((n-1)*10) + 4" text-anchor="end" class="svg-label">
              {{ ((n-1)*100000).toLocaleString('pt-BR') }},00
            </text>

            <g v-for="marca in marcas" :key="marca.nome">
              <g v-if="marcasAtivas.includes(marca.nome)">
                <polyline :points="gerarPontos(marca.nome)" fill="none" :stroke="marca.cor" stroke-width="2.5" />
                <circle v-for="(val, idx) in dadosVendasOriginal[marca.nome].slice(mesInicio, parseInt(mesFim) + 1)" :key="idx"
                  :cx="getX(idx)" :cy="getY(val)" r="4" :fill="marca.cor" stroke="white" stroke-width="2" />
              </g>
            </g>

            <text v-for="(mes, i) in mesesFiltrados" :key="mes" :x="getX(i)" :y="alturaSVG - 10" text-anchor="middle" class="svg-label-x">
              {{ mes }}
            </text>
          </svg>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Dashboard Base */
.pbi-dashboard { background-color: #f3f3f3; height: 100vh; display: flex; flex-direction: column; font-family: 'Segoe UI', sans-serif; overflow: hidden; }

/* Filtros */
.pbi-header { background: white; padding: 10px 15px; display: flex; flex-wrap: wrap; align-items: center; gap: 15px; font-size: 11px; font-weight: bold; border-bottom: 1px solid #ddd; z-index: 100; }
.f-group { display: flex; align-items: center; gap: 6px; }
.f-group select { border: 1px solid #ccc; padding: 2px 5px; border-radius: 2px; }
.pbi-tabs { border: 1px solid #ccc; border-radius: 2px; display: flex; }
.pbi-tabs button { border: none; background: white; padding: 2px 10px; font-size: 10px; cursor: pointer; color: #777; }
.pbi-tabs button.active { background: #eee; color: #000; font-weight: bold; }
.btn-pbi-red { background: #ff4d4d; color: white; border: none; padding: 6px 15px; border-radius: 6px; font-weight: bold; margin-left: auto; cursor: pointer; }

/* Conteúdo */
.pbi-content { flex: 1; padding: 12px; display: flex; flex-direction: column; gap: 12px; overflow: hidden; }
.top-row { display: grid; grid-template-columns: 3fr 1fr 1fr; gap: 12px; height: 35%; min-height: 220px; }
.kpi-grid { display: grid; grid-template-columns: repeat(3, 1fr); grid-template-rows: repeat(2, 1fr); gap: 10px; }

.pbi-card { background: white; border-radius: 12px; border: 1px solid #e5e5e5; box-shadow: 0 2px 5px rgba(0,0,0,0.03); overflow: hidden; }

/* KPI Styling */
.kpi-card { padding: 10px; position: relative; }
.kpi-inner { display: flex; justify-content: space-between; align-items: center; height: 100%; }
.kpi-text { flex: 1; margin: 0 8px; }
.kpi-label { font-size: 9px; font-weight: 800; color: #222; text-transform: uppercase; }
.kpi-val { font-size: 20px; font-weight: 800; color: #000; margin: 2px 0; }
.kpi-sub { font-size: 10px; display: flex; align-items: center; gap: 4px; font-weight: 600; color: #666; }
.pbi-dot { color: #228b22; font-size: 12px; }
.pbi-perc { color: #228b22; }
.kpi-icon.mirror { opacity: 0.05; transform: scaleX(-1); }

/* Venda Anual e Segmento */
.side-chart { padding: 12px; display: flex; flex-direction: column; }
.chart-header { font-size: 11px; font-weight: bold; color: #444; margin-bottom: 5px; }
.venda-anual-bars { flex: 1; display: flex; align-items: flex-end; justify-content: space-between; padding: 10px 5px; }
.va-col { flex: 1; display: flex; flex-direction: column; align-items: center; height: 100%; justify-content: flex-end; }
.va-bar { background-color: #f58220; width: 60%; border-radius: 2px 2px 0 0; min-width: 8px; }
.va-bar.last-bar { background-color: #fcd5b5; }
.va-txt { font-size: 8px; font-weight: bold; color: #333; margin-bottom: 3px; }
.va-year { font-size: 9px; color: #999; margin-top: 4px; }

.seg-viz { flex: 1; display: flex; justify-content: space-around; align-items: flex-end; padding-bottom: 5px; }
.seg-bar-group { display: flex; flex-direction: column; align-items: center; }

/* Gráfico de Linhas Interativo */
.main-viz-card { flex: 1; display: flex; flex-direction: column; padding: 15px; }
.chart-header-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; }
.chart-title { font-size: 14px; font-weight: bold; color: #333; }

.pbi-line-legend { display: flex; flex-wrap: wrap; gap: 12px; justify-content: flex-end; max-width: 70%; }
.leg-item { display: flex; align-items: center; gap: 4px; cursor: pointer; transition: 0.2s; }
.leg-item.inactive { opacity: 0.2; }
.leg-line { width: 12px; height: 2px; }
.leg-dot { width: 7px; height: 7px; border-radius: 50%; background: white; border: 2px solid; margin-left: -10px; }
.leg-text { font-size: 9px; font-weight: bold; color: #666; text-transform: uppercase; }

.main-viz-wrapper { flex: 1; position: relative; width: 100%; }
.svg-line-chart { width: 100%; height: 100%; overflow: visible; }
.svg-label { font-size: 9px; fill: #999; }
.svg-label-x { font-size: 10px; fill: #555; font-weight: bold; }

.font-bold { font-weight: bold; }
.text-center { text-align: center; }
</style>
