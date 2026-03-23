<script setup>
import { ref, computed } from 'vue';

// --- DADOS DO GRÁFICO DE LINHAS (LUBRIFICANTES) ---
const marcas = [
  { nome: 'YPF', cor: '#0070c0' },
  { nome: 'VIBRA', cor: '#004415' },
  { nome: 'VALVOLINE', cor: '#00acee' },
  { nome: 'TOTALENERGIES', cor: '#ff0000' },
  { nome: 'RAIZEN', cor: '#990093' },
  { nome: 'PETRONAS', cor: '#00a19c' },
  { nome: 'MOOVE', cor: '#85bd00' },
  { nome: 'LWART', cor: '#006cb7' },
  { nome: 'ICONIC', cor: '#ff511f' },
  { nome: 'CASTROL', cor: '#f00023' }
];

const mesesLista = ['Jan', 'Fev', 'Mar', 'Abr', 'Mai', 'Jun', 'Jul', 'Ago', 'Set', 'Out', 'Nov', 'Dez'];

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

// --- CONFIGURAÇÃO DO GRÁFICO ---
const larguraSVG = 1000;
const alturaSVG = 400; 
const paddingTop = 40;    // Espaço no topo para não cortar a linha de 100M
const paddingBottom = 70; // Espaço na base para o "0" e os meses
const paddingLeft = 110;  // Espaço para os números 100.000.000,00
const paddingRight = 30;

const maxYData = 50; // O topo (100M)

const mesesFiltrados = computed(() => mesesLista.slice(mesInicio.value, parseInt(mesFim.value) + 1));

const getX = (indexRelativo) => {
  const qtd = mesesFiltrados.value.length;
  const areaUtilX = larguraSVG - paddingLeft - paddingRight;
  return paddingLeft + (indexRelativo * areaUtilX / (qtd - 1));
};

const getY = (val) => {
  const areaUtilY = alturaSVG - paddingTop - paddingBottom;
  // O valor "0" ficará exatamente na linha do paddingBottom
  return (alturaSVG - paddingBottom) - (val / maxYData * areaUtilY);
};

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
      <div class="top-row">
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

      <!-- GRÁFICO INFERIOR CORRIGIDO -->
      <div class="pbi-card main-viz-card">
        <div class="chart-header-row">
          <span class="chart-title">Vendas de Lubrificantes por mês</span>
          <div class="pbi-line-legend">
            <div v-for="marca in marcas" :key="marca.nome" 
                 class="leg-item" 
                 :class="{ 'inactive': !marcasAtivas.includes(marca.nome) }"
                 @click="toggleMarca(marca.nome)">
              <img :src="`/logos/lubrificantes/${marca.nome.toLowerCase()}.png`" 
                   :alt="marca.nome" 
                   class="brand-logo" />
            </div>
          </div>
        </div>
        
        <div class="main-viz-wrapper">
          <svg class="svg-line-chart" :viewBox="`0 0 ${larguraSVG} ${alturaSVG}`" preserveAspectRatio="xMidYMid meet">
            <!-- LINHAS DE GRADE (Y) -->
            <line v-for="n in 6" :key="'grid'+n" 
              :x1="paddingLeft" :y1="getY((n-1)*10)" :x2="larguraSVG - paddingRight" :y2="getY((n-1)*10)" 
              stroke="#eeeeee" stroke-width="1" />
            
            <!-- LABELS DO EIXO Y (Garante o 0 até 100M) -->
            <text v-for="n in 6" :key="'labelY'+n" 
              :x="paddingLeft - 15" :y="getY((n-1)*10) + 4" 
              text-anchor="end" class="svg-label">
              {{ ((n-1)*20000000).toLocaleString('pt-BR') }},00
            </text>

            <!-- AS LINHAS DOS DADOS -->
            <g v-for="marca in marcas" :key="'linegroup'+marca.nome">
              <g v-if="marcasAtivas.includes(marca.nome)">
                <polyline :points="gerarPontos(marca.nome)" fill="none" :stroke="marca.cor" stroke-width="3" stroke-linejoin="round" />
                <circle v-for="(val, idx) in dadosVendasOriginal[marca.nome].slice(mesInicio, parseInt(mesFim) + 1)" :key="'dot'+idx"
                  :cx="getX(idx)" :cy="getY(val)" r="4.5" :fill="marca.cor" stroke="white" stroke-width="1.5" />
              </g>
            </g>

            <!-- MESES (EIXO X) -->
            <text v-for="(mes, i) in mesesFiltrados" :key="'labelX'+mes" 
              :x="getX(i)" :y="alturaSVG - 25" 
              text-anchor="middle" class="svg-label-x">
              {{ mes }}
            </text>
          </svg>
        </div>
      </div>
    </div>
    
  </div>
</template>

<style scoped>
.pbi-dashboard { background-color: #f3f3f3; height: 100vh; display: flex; flex-direction: column; font-family: 'Segoe UI', sans-serif; overflow: hidden; }

/* Header e Filtros */
.pbi-header { background: white; padding: 10px 15px; display: flex; flex-wrap: wrap; align-items: center; gap: 15px; font-size: 11px; font-weight: bold; border-bottom: 1px solid #ddd; }
.f-group { display: flex; align-items: center; gap: 6px; }
.f-group select { border: 1px solid #ccc; padding: 2px 5px; border-radius: 2px; }
.pbi-tabs { border: 1px solid #ccc; border-radius: 2px; display: flex; }
.pbi-tabs button { border: none; background: white; padding: 2px 10px; font-size: 10px; cursor: pointer; color: #777; }
.pbi-tabs button.active { background: #eee; color: #000; font-weight: bold; }
.btn-pbi-red { background: #ff4d4d; color: white; border: none; padding: 6px 15px; border-radius: 6px; font-weight: bold; margin-left: auto; cursor: pointer; }

/* Layout Geral */
.pbi-content { flex: 1; padding: 15px; display: flex; flex-direction: column; gap: 15px; overflow: hidden; }
.top-row { display: grid; grid-template-columns: 3fr 1fr 1fr; gap: 15px; height: 30%; min-height: 200px; }
.kpi-grid { display: grid; grid-template-columns: repeat(3, 1fr); grid-template-rows: repeat(2, 1fr); gap: 12px; }

.pbi-card { background: white; border-radius: 12px; border: 1px solid #e5e5e5; box-shadow: 0 2px 5px rgba(0,0,0,0.03); overflow: hidden; }

/* KPIs */
.kpi-card { padding: 10px; }
.kpi-inner { display: flex; justify-content: space-between; align-items: center; height: 100%; }
.kpi-label { font-size: 9px; font-weight: 800; color: #444; text-transform: uppercase; }
.kpi-val { font-size: 20px; font-weight: 800; color: #000; margin: 2px 0; }
.kpi-sub { font-size: 10px; display: flex; align-items: center; gap: 4px; color: #666; font-weight: 600; }
.pbi-dot { color: #228b22; }
.pbi-perc { color: #228b22; }
.kpi-icon.mirror { opacity: 0.05; transform: scaleX(-1); }

/* Venda Anual */
.side-chart { padding: 12px; display: flex; flex-direction: column; }
.chart-header { font-size: 11px; font-weight: bold; color: #444; margin-bottom: 8px; }
.venda-anual-bars { flex: 1; display: flex; align-items: flex-end; justify-content: space-between; padding: 5px; }
.va-col { flex: 1; display: flex; flex-direction: column; align-items: center; height: 100%; justify-content: flex-end; }
.va-bar { background-color: #f58220; width: 60%; border-radius: 2px; min-width: 6px; }
.va-txt { font-size: 8px; font-weight: bold; margin-bottom: 2px; }
.va-year { font-size: 9px; color: #999; margin-top: 4px; }
.seg-viz { flex: 1; display: flex; justify-content: space-around; align-items: flex-end; }

/* O GRÁFICO PRINCIPAL */
.main-viz-card { flex: 1; display: flex; flex-direction: column; padding: 20px; }
.chart-header-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; }
.chart-title { font-size: 16px; font-weight: bold; color: #333; }

.pbi-line-legend { display: flex; flex-wrap: wrap; gap: 12px; justify-content: flex-end; max-width: 75%; }
.leg-item { cursor: pointer; transition: 0.2s; }
.leg-item.inactive { opacity: 0.15; filter: grayscale(1); }
.brand-logo { height: 64px; width: auto; object-fit: contain; }

.main-viz-wrapper { flex: 1; position: relative; width: 100%; overflow: hidden; }
.svg-line-chart { width: 100%; height: 100%; display: block; }

.svg-label { font-size: 10px; fill: #888; font-weight: bold; }
.svg-label-x { font-size: 12px; fill: #444; font-weight: 800; }

.pbi-footer { text-align: center; padding: 10px; font-size: 11px; color: #777; background: white; border-top: 1px solid #eee; }

.font-bold { font-weight: bold; }
.text-center { text-align: center; }
</style>
