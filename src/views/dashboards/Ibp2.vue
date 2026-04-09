<script setup>
import { ref, computed } from 'vue';

// --- DADOS ---
const marcas = [
  { nome: 'TotalEnergies', cor: '#f58220', share: '32,1%', logo: 'https://upload.wikimedia.org/wikipedia/fr/thumb/f/f7/TotalEnergies_logo.svg/1024px-TotalEnergies_logo.svg.png' },
  { nome: 'Lubrax', cor: '#009543', share: '24,8%', logo: 'https://vignette.wikia.nocookie.net/logopedia/images/b/b3/Lubrax.png' },
  { nome: 'Mobil', cor: '#003087', share: '15,2%', logo: 'https://upload.wikimedia.org/wikipedia/commons/thumb/8/80/Mobil_Logo.svg/1200px-Mobil_Logo.svg.png' },
  { nome: 'Shell', cor: '#fbce07', share: '11,4%', logo: 'https://upload.wikimedia.org/wikipedia/pt/thumb/d/d3/Logo_Shell.svg/1200px-Logo_Shell.svg.png' },
  { nome: 'Castrol', cor: '#008144', share: '8,9%', logo: 'https://upload.wikimedia.org/wikipedia/pt/thumb/a/ae/Logo_Castrol.svg/1200px-Logo_Castrol.svg.png' },
  { nome: 'Valvoline', cor: '#da1f28', share: '5,6%', logo: 'https://upload.wikimedia.org/wikipedia/commons/thumb/7/77/Valvoline_Logo.svg/1200px-Valvoline_Logo.svg.png' },
];

const mesesLista = ['Jan', 'Fev', 'Mar', 'Abr', 'Mai', 'Jun', 'Jul', 'Ago', 'Set', 'Out', 'Nov', 'Dez'];

// Dados simulados baseados no print (Valores em R$ ou Volume)
const mediaMercado = [400, 380, 405, 399, 410, 420, 410, 410, 410, 460, 415, 375];
const vendasTotal = [330, 320, 345, 335, 355, 350, 345, 350, 345, 370, 350, 330];
const vendasConcorrente1 = [300, 280, 310, 305, 315, 320, 320, 325, 315, 330, 320, 310];
const vendasConcorrente2 = [270, 260, 280, 275, 290, 295, 295, 290, 295, 305, 295, 285];

// --- CONFIGURAÇÃO DO GRÁFICO SVG ---
const larguraSVG = 1000;
const alturaSVG = 400;
const padding = { top: 40, bottom: 40, left: 50, right: 20 };

const getX = (idx) => padding.left + (idx * (larguraSVG - padding.left - padding.right) / 11);
const getY = (val) => (alturaSVG - padding.bottom) - (val / 500 * (alturaSVG - padding.top - padding.bottom));

const gerarPontos = (dados) => dados.map((v, i) => `${getX(i)},${getY(v)}`).join(' ');

const kpis = [
    { label: 'ANÁLISE MERCADO - ANUAL', val: '1.230.987 m³', sub: '1.216.760 m³', perc: '+1,17%', color: '#e7f7ed' },
    { label: 'ANÁLISE MERCADO - MENSAL', val: '1.230.987 m³', sub: '1.216.760 m³', perc: '+1,17%', color: '#e7f7ed' },
    { label: 'ANÁLISE MERCADO - PROJEÇÃO', val: '1.230.987 m³', sub: '1.233.992 m³', perc: '+1,17%', color: '#e7f7ed' },
    { label: 'TotalEnergies - ANUAL', val: '16.617 m³', sub: '15.826 m³', perc: '+5,00%', color: '#fff4e5' },
    { label: 'TotalEnergies - MENSAL', val: '16.617 m³', sub: '15.826 m³', perc: '+5,00%', color: '#e9f2ff' },
    { label: 'TotalEnergies - PROJEÇÃO', val: '16.617 m³', sub: '15.826 m³', perc: '+5,00%', color: '#f0f0f0' },
];

const histAnual = [
    {y:'2017', v:'21.154'}, {y:'2018', v:'24.500'}, {y:'2019', v:'24.100'}, 
    {y:'2020', v:'15.004'}, {y:'2021', v:'28.984'}, {y:'2022', v:'30.354'}, 
    {y:'2023', v:'31.200'}, {y:'2024', v:'28.700'}, {y:'2025', v:'35.445', act: true}
];
</script>

<template>
  <div class="pbi-dashboard">
    <!-- HEADER FILTROS -->
    <header class="pbi-header">
      <div class="filter-group">
        <select class="pbi-select main-sel"><option>TotalEnergies</option></select>
        <div class="segment-toggle">
            <button>CONSUMIDOR</button>
            <button class="active">REVENDEDOR</button>
        </div>
        <div class="date-picker">
            <span class="icon">📅</span> 02/17 – 31/12/2025
        </div>
      </div>
      <button class="btn-clear">Limpar Filtros</button>
    </header>

    <div class="pbi-content">
      <!-- TOP ROW: KPIs E GRÁFICOS PEQUENOS -->
      <div class="top-layout">
        <div class="kpi-grid">
          <div class="kpi-card" v-for="(k, i) in kpis" :key="i" :style="{ backgroundColor: k.color }">
            <div class="kpi-label">{{ k.label }}</div>
            <div class="kpi-val">{{ k.val }}</div>
            <div class="kpi-sub">
                {{ k.sub }} <span class="perc-tag">{{ k.perc }}</span>
            </div>
          </div>
        </div>

        <div class="side-card">
          <div class="card-title">Venda Anual <span class="download-icon">↓</span></div>
          <div class="anual-chart">
            <div v-for="h in histAnual" :key="h.y" class="bar-col">
              <span class="bar-val">{{ h.v }}</span>
              <div class="bar" :class="{ 'active-bar': h.act }" :style="{ height: (parseInt(h.v.replace('.',''))/400) + 'px' }"></div>
              <span class="bar-year">{{ h.y }}</span>
            </div>
          </div>
        </div>

        <div class="side-card">
          <div class="card-title">Vendas por Segmento <span class="download-icon">↓</span></div>
          <div class="segment-bars">
            <div class="seg-col">
                <span class="bar-val">35.445</span>
                <div class="bar yellow" style="height: 100px;"></div>
                <span class="bar-year">Revendedor</span>
            </div>
            <div class="seg-col">
                <span class="bar-val">35.445</span>
                <div class="bar blue" style="height: 50px;"></div>
                <span class="bar-year">Consumidor</span>
            </div>
          </div>
        </div>
      </div>

      <!-- MAIN CHART AREA -->
      <div class="main-viz-container">
        <div class="main-chart-card">
          <div class="main-chart-header">
            <div>
                <h3>Vendas de Lubrificantes por Mês</h3>
                <p class="subtitle">Barras = Média de Mercado (contexto) • Linhas = Volume por Marca</p>
            </div>
            <div class="year-filters">
                <button>2023</button><button>2024</button><button class="active">2025</button>
                <button class="btn-export">↓ Exportar</button>
            </div>
          </div>

          <div class="chart-content">
            <svg class="main-svg" :viewBox="`0 0 ${larguraSVG} ${alturaSVG}`">
                <!-- GRID LINES -->
                <line v-for="h in [100, 200, 300, 400, 500]" :key="h" 
                      x1="50" :y1="getY(h)" x2="980" :y2="getY(h)" stroke="#f0f0f0" />
                
                <!-- BARS (Market Context) -->
                <g v-for="(v, i) in mediaMercado" :key="'bar'+i">
                    <rect :x="getX(i) - 25" :y="getY(v)" width="50" :height="getY(0) - getY(v)" 
                          :fill="i === 9 ? '#ffbb55' : '#e9ecef'" rx="2" />
                    <text :x="getX(i)" :y="getY(v) - 10" text-anchor="middle" class="bar-label">R$ {{ v }}.000</text>
                </g>

                <!-- LINES (Brands) -->
                <polyline :points="gerarPontos(vendasTotal)" fill="none" stroke="#f58220" stroke-width="3" />
                <circle v-for="(v, i) in vendasTotal" :key="'c1'+i" :cx="getX(i)" :cy="getY(v)" r="4" fill="#f58220" />

                <polyline :points="gerarPontos(vendasConcorrente1)" fill="none" stroke="#5a6772" stroke-width="1.5" />
                <polyline :points="gerarPontos(vendasConcorrente2)" fill="none" stroke="#adb5bd" stroke-width="1.5" />

                <!-- X AXIS -->
                <text v-for="(m, i) in mesesLista" :key="m" :x="getX(i)" :y="alturaSVG - 10" text-anchor="middle" class="axis-label">{{ m }}</text>
            </svg>
          </div>
        </div>

        <!-- RIGHT LEGEND SHARE -->
        <div class="share-legend">
            <div class="share-header">% Share</div>
            <div v-for="m in marcas" :key="m.nome" class="share-item">
                <img :src="m.logo" class="mini-logo">
                <span class="share-name">{{ m.nome }}</span>
                <span class="share-pill" :style="{ backgroundColor: m.nome === 'TotalEnergies' ? '#f58220' : '#e9ecef', color: m.nome === 'TotalEnergies' ? 'white' : '#666' }">
                    {{ m.share }}
                </span>
            </div>
        </div>
      </div>

      <!-- FOOTER IA -->
      <footer class="ia-footer">
        <div class="ia-icon">✦</div>
        <div class="ia-text">
            <strong>Dashboard Legenda por IA</strong> <span class="ia-tag">LubAssist IA</span>
            <p>Este dashboard apresenta análise de market share (IBP). Os 6 cards superiores comparam o mercado total com a TotalEnergies (em laranja). O gráfico central sobrepõe barras cinzas (média de mercado) com linhas por marca. Pico sazonal observado entre Maio e Outubro.</p>
        </div>
      </footer>
    </div>
  </div>
</template>

<style scoped>
.pbi-dashboard { background: #f4f4f4; height: 100vh; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; display: flex; flex-direction: column; color: #333; }

/* HEADER */
.pbi-header { background: white; padding: 12px 20px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #ddd; }
.filter-group { display: flex; gap: 15px; align-items: center; }
.pbi-select { padding: 5px 15px; border: 1px solid #ddd; border-radius: 4px; font-weight: 600; }
.segment-toggle { display: flex; background: #eee; padding: 3px; border-radius: 4px; }
.segment-toggle button { border: none; background: transparent; padding: 4px 12px; font-size: 11px; font-weight: bold; cursor: pointer; color: #888; }
.segment-toggle button.active { background: #f58220; color: white; border-radius: 3px; }
.date-picker { background: #f9f9f9; border: 1px solid #ddd; padding: 5px 15px; border-radius: 4px; font-size: 13px; }
.btn-clear { background: #f58220; color: white; border: none; padding: 8px 20px; border-radius: 4px; font-weight: bold; cursor: pointer; }

/* CONTENT LAYOUT */
.pbi-content { padding: 15px; flex: 1; display: flex; flex-direction: column; gap: 15px; }
.top-layout { display: grid; grid-template-columns: 2fr 1fr 1fr; gap: 15px; height: 200px; }

/* KPIs */
.kpi-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; }
.kpi-card { padding: 15px; border-radius: 8px; display: flex; flex-direction: column; justify-content: center; border: 1px solid rgba(0,0,0,0.05); }
.kpi-label { font-size: 10px; font-weight: bold; color: #666; margin-bottom: 5px; text-transform: uppercase; }
.kpi-val { font-size: 22px; font-weight: 900; }
.kpi-sub { font-size: 11px; margin-top: 5px; color: #777; }
.perc-tag { color: #28a745; font-weight: bold; margin-left: 5px; }

/* SMALL CHARTS */
.side-card { background: white; border-radius: 8px; padding: 12px; border: 1px solid #ddd; }
.card-title { font-size: 12px; font-weight: bold; margin-bottom: 10px; display: flex; justify-content: space-between; }
.anual-chart, .segment-bars { display: flex; align-items: flex-end; justify-content: space-between; height: 130px; padding-bottom: 20px; }
.bar-col, .seg-col { display: flex; flex-direction: column; align-items: center; width: 100%; }
.bar { background: #adb5bd; width: 60%; border-radius: 2px 2px 0 0; transition: 0.3s; }
.active-bar { background: #f58220; }
.yellow { background: #d4af37; }
.blue { background: #3b82f6; }
.bar-val { font-size: 9px; font-weight: bold; margin-bottom: 3px; }
.bar-year { font-size: 9px; color: #999; margin-top: 5px; text-align: center; }

/* MAIN VIZ AREA */
.main-viz-container { flex: 1; display: grid; grid-template-columns: 1fr 180px; gap: 15px; min-height: 450px; }
.main-chart-card { background: white; border-radius: 8px; border: 1px solid #ddd; padding: 20px; display: flex; flex-direction: column; }
.main-chart-header { display: flex; justify-content: space-between; margin-bottom: 20px; }
.main-chart-header h3 { margin: 0; font-size: 18px; }
.subtitle { font-size: 12px; color: #888; margin: 4px 0 0 0; }
.year-filters button { border: 1px solid #ddd; background: #f9f9f9; padding: 5px 12px; cursor: pointer; font-size: 12px; }
.year-filters button.active { background: #f58220; color: white; border-color: #f58220; }
.btn-export { margin-left: 10px; border-radius: 4px; }

.chart-content { flex: 1; position: relative; }
.main-svg { width: 100%; height: 100%; overflow: visible; }
.bar-label { font-size: 10px; fill: #444; font-weight: bold; }
.axis-label { font-size: 11px; fill: #888; font-weight: bold; }

/* LEGEND SHARE */
.share-legend { background: white; border-radius: 8px; border: 1px solid #ddd; padding: 15px; }
.share-header { font-size: 12px; font-weight: bold; color: #888; margin-bottom: 15px; text-align: right; }
.share-item { display: flex; align-items: center; margin-bottom: 15px; font-size: 11px; }
.mini-logo { width: 24px; height: 24px; object-fit: contain; margin-right: 8px; }
.share-name { flex: 1; font-weight: 600; color: #555; }
.share-pill { font-size: 10px; font-weight: bold; padding: 2px 8px; border-radius: 10px; }

/* FOOTER IA */
.ia-footer { background: #1a1a1b; color: #ccc; border-radius: 8px; padding: 20px; display: flex; gap: 20px; align-items: flex-start; }
.ia-icon { background: linear-gradient(45deg, #f58220, #ffd000); -webkit-background-clip: text; color: transparent; font-size: 24px; }
.ia-text p { margin: 10px 0 0 0; font-size: 13px; line-height: 1.4; color: #999; }
.ia-tag { background: rgba(245, 130, 32, 0.2); color: #f58220; font-size: 10px; padding: 2px 6px; border-radius: 4px; margin-left: 10px; }
</style>
