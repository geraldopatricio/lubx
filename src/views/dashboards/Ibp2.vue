<script setup>
import { ref, computed } from 'vue';

// --- CONFIGURAÇÕES E CORES ---
const brandColor = '#f58220'; // Laranja TotalEnergies
const accentColor = '#32a1ff'; // Azul secundário

const marcas = [
  { nome: 'TotalEnergies', cor: '#f58220', share: '32,1%', logo: 'totalenergies' },
  { nome: 'Lubrax', cor: '#009541', share: '24,8%', logo: 'lubrax' },
  { nome: 'Mobil', cor: '#00438a', share: '15,2%', logo: 'mobil' },
  { nome: 'Shell', cor: '#fbce07', share: '11,4%', logo: 'shell' },
  { nome: 'Castrol', cor: '#00833d', share: '8,9%', logo: 'castrol' },
  { nome: 'Valvoline', cor: '#0055a4', share: '5,6%', logo: 'valvoline' },
  { nome: 'Outros', cor: '#999', share: '2,0%', logo: 'outros' }
];

const mesesLista = ['Jan', 'Fev', 'Mar', 'Abr', 'Mai', 'Jun', 'Jul', 'Ago', 'Set', 'Out', 'Nov', 'Dez'];
const marcasAtivas = ref(marcas.map(m => m.nome));
const anoSelecionado = ref(2025);

// Dados fictícios baseados no print
const marketAvgData = [400, 380, 405, 399, 410, 420, 410, 410, 410, 460, 415, 375]; // Valores em "milhões" para as barras cinzas
const totalEnergiesLine = [320, 305, 335, 315, 345, 330, 335, 325, 320, 350, 330, 310];

// --- KPIs ---
const kpis = [
    { label: 'ANÁLISE MERCADO - ANUAL', val: '1.230.987 m³', sub: '1.216.760 m³', perc: '+1,17%', color: 'green' },
    { label: 'ANÁLISE MERCADO - MENSAL', val: '1.230.987 m³', sub: '1.216.760 m³', perc: '+1,17%', color: 'green' },
    { label: 'ANÁLISE MERCADO - PROJEÇÃO', val: '1.230.987 m³', sub: '1.233.992 m³', perc: '+1,17%', color: 'green' },
    { label: 'TotalEnergies - ANUAL', val: '16.617 m³', sub: '15.826 m³', perc: '+5,00%', color: 'green', bg: '#fff6ed' },
    { label: 'TotalEnergies - MENSAL', val: '16.617 m³', sub: '15.826 m³', perc: '+5,00%', color: 'green', bg: '#eef6ff' },
    { label: 'TotalEnergies - PROJEÇÃO', val: '16.617 m³', sub: '15.826 m³', perc: '+5,00%', color: 'green', bg: '#e5e5e5' },
];

const vendaAnualData = [
    {y:'2017', v:'21.154'}, {y:'2018', v:'24.500'}, {y:'2019', v:'24.100'}, 
    {y:'2020', v:'15.004'}, {y:'2021', v:'28.984'}, {y:'2022', v:'30.354'}, 
    {y:'2023', v:'31.200'}, {y:'2024', v:'28.700'}, {y:'2025', v:'35.445', active: true}
];

// Funções de auxílio para o SVG
const getX = (i) => 50 + (i * 80);
const getYBar = (val) => 350 - (val * 0.6);
const getYLine = (val) => 330 - (val * 0.6);

</script>

<template>
  <div class="dashboard-container">
    <!-- HEADER FILTROS -->
    <header class="filter-bar">
      <div class="filter-group">
        <select class="pbi-select main-brand"><option>TotalEnergies</option></select>
      </div>
      
      <div class="segment-toggle">
        <button>CONSUMIDOR</button>
        <button class="active">REVENDEDOR</button>
      </div>

      <div class="date-picker">
        <span class="icon">📅</span>
        <input type="text" value="02/17 – 31/12/2025" readonly />
        <span class="arrow">▼</span>
      </div>

      <button class="btn-clear-filters">Limpar Filtros</button>
    </header>

    <main class="content-grid">
      <!-- ROW 1: KPIs -->
      <div class="kpi-row">
        <div v-for="(k, i) in kpis" :key="i" class="kpi-card" :style="{ backgroundColor: k.bg || 'white' }">
          <div class="kpi-header">{{ k.label }}</div>
          <div class="kpi-value">{{ k.val }}</div>
          <div class="kpi-footer">
            <span class="sub-val">{{ k.sub }}</span>
            <span class="perc-tag">{{ k.perc }}</span>
          </div>
        </div>
      </div>

      <!-- ROW 2: SMALL CHARTS -->
      <div class="charts-row">
        <div class="pbi-card chart-container">
          <div class="card-header">
            <span>Venda Anual</span>
            <span class="icon-down">⭳</span>
          </div>
          <div class="bar-chart-annual">
            <div v-for="item in vendaAnualData" :key="item.y" class="bar-item">
              <span class="bar-val">{{ item.v }}</span>
              <div class="bar" :class="{ 'orange': item.active }" :style="{ height: (parseInt(item.v)/35000 * 100) + '%' }"></div>
              <span class="bar-label">{{ item.y }}</span>
            </div>
          </div>
        </div>

        <div class="pbi-card chart-container">
          <div class="card-header">
            <span>Vendas por Segmento</span>
            <span class="icon-down">⭳</span>
          </div>
          <div class="segment-viz">
            <div class="seg-item">
               <span class="bar-val">35.445</span>
               <div class="bar yellow" style="height: 80%"></div>
               <span class="bar-label">Revendedor</span>
            </div>
            <div class="seg-item">
               <span class="bar-val">35.445</span>
               <div class="bar blue" style="height: 60%"></div>
               <span class="bar-label">Consumidor</span>
            </div>
          </div>
        </div>
      </div>

      <!-- ROW 3: MAIN CHART -->
      <div class="pbi-card main-viz">
        <div class="main-viz-header">
          <div class="title-area">
            <h3>Vendas de Lubrificantes por Mês</h3>
            <p>Barras = Média de Mercado (contexto) • Linhas = Volume por Marca • Coluna % = Share de Venda</p>
          </div>
          <div class="main-filters">
            <div class="year-selector">
              <button>2023</button><button>2024</button><button class="active">2025</button>
            </div>
            <button class="btn-export">⭳ Exportar</button>
          </div>
        </div>

        <div class="legend-brands">
          <div v-for="m in marcas" :key="m.nome" class="leg-brand">
            <span class="line-dot" :style="{ background: m.cor }"></span>
            {{ m.nome }}
          </div>
        </div>

        <div class="chart-content-wrapper">
          <!-- SVG CHART AREA -->
          <div class="svg-container">
            <svg viewBox="0 0 1000 400" preserveAspectRatio="xMidYMid meet">
              <!-- Grid Lines -->
              <line v-for="n in 5" :x1="40" :y1="350 - (n*70)" x2="960" :y2="350 - (n*70)" stroke="#e0e0e0" />
              
              <!-- Eixo Y labels -->
              <text v-for="n in 6" :x="35" :y="355 - ((n-1)*70)" text-anchor="end" class="y-label">{{ (n-1)*100 }}M</text>

              <!-- Barras de Contexto -->
              <g v-for="(v, i) in marketAvgData" :key="'bar'+i">
                <rect :x="getX(i) - 20" :y="getYBar(v)" width="40" :height="350 - getYBar(v)" 
                      :fill="i === 9 ? '#ffb366' : '#e5e7eb'" rx="2" />
                <text :x="getX(i)" :y="getYBar(v) - 10" text-anchor="middle" class="bar-text">R$ {{ v.toFixed(3) }}</text>
              </g>

              <!-- Linha TotalEnergies (Destaque) -->
              <polyline :points="totalEnergiesLine.map((v, i) => `${getX(i)},${getYLine(v)}`).join(' ')" 
                        fill="none" stroke="#f58220" stroke-width="4" />
              <circle v-for="(v, i) in totalEnergiesLine" :cx="getX(i)" :cy="getYLine(v)" r="5" fill="#f58220" />

              <!-- Linhas Outros (Exemplo simplificado) -->
              <polyline :points="marketAvgData.map((v, i) => `${getX(i)},${getYLine(v-100)}`).join(' ')" 
                        fill="none" stroke="#666" stroke-width="1.5" opacity="0.4" />

              <!-- Eixo X Meses -->
              <text v-for="(m, i) in mesesLista" :x="getX(i)" y="380" text-anchor="middle" class="x-label">{{ m }}</text>
            </svg>
          </div>

          <!-- SIDE SHARE COLUMN -->
          <div class="share-column">
            <div class="share-header">% Share</div>
            <div v-for="m in marcas" :key="'share'+m.nome" class="share-item">
              <img :src="`/logos/lubrificantes/${m.logo}.png`" :alt="m.nome" class="brand-logo-mini" />
              <span class="share-name">{{ m.nome }}</span>
              <span class="share-badge" :style="{ backgroundColor: m.nome === 'TotalEnergies' ? '#f58220' : '#e5e7eb', color: m.nome === 'TotalEnergies' ? 'white' : '#555' }">
                {{ m.share }}
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- FOOTER AI LEGEND -->
      <footer class="ai-legend-card">
        <div class="ai-icon">✦</div>
        <div class="ai-content">
          <div class="ai-title">Dashboard Legenda por IA <span class="badge-ia">LubAssist IA</span></div>
          <p>Este dashboard apresenta análise de market share (IBP). Os 6 cards superiores comparam o mercado total com a TotalEnergies (em laranja) nas visões anual, mensal e projetada. O gráfico central sobrepõe barras cinzas (média de mercado como contexto) com linhas por marca — laranja para TotalEnergies, tons de cinza para concorrentes. A coluna direita exibe o % de share de cada marca. Pico sazonal observado entre Maio e Outubro.</p>
        </div>
      </footer>
    </main>
  </div>
</template>

<style scoped>
.dashboard-container {
  background-color: #f3f3f3;
  min-height: 100vh;
  padding: 10px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  display: flex;
  flex-direction: column;
  gap: 10px;
  color: #333;
}

/* --- HEADER / FILTROS --- */
.filter-bar {
  display: flex;
  align-items: center;
  gap: 15px;
  background: white;
  padding: 8px 15px;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.pbi-select {
  border: 1px solid #ddd;
  padding: 5px 30px 5px 10px;
  border-radius: 4px;
  appearance: none;
  background: url("data:image/svg+xml;charset=UTF-8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='currentColor' stroke-width='3' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='M6 9l6 6 6-6'%3E%3C/path%3E%3C/svg%3E") no-repeat right 10px center;
  color: #d84b16;
  font-weight: bold;
}

.segment-toggle {
  display: flex;
  background: #eee;
  padding: 3px;
  border-radius: 6px;
}
.segment-toggle button {
  border: none;
  background: transparent;
  padding: 4px 12px;
  font-size: 11px;
  font-weight: bold;
  cursor: pointer;
  color: #777;
}
.segment-toggle button.active {
  background: #f58220;
  color: white;
  border-radius: 4px;
}

.date-picker {
  display: flex;
  align-items: center;
  border: 1px solid #ddd;
  padding: 4px 10px;
  border-radius: 4px;
  background: white;
  font-size: 12px;
}
.date-picker input { border: none; outline: none; margin: 0 5px; width: 130px; }

.btn-clear-filters {
  margin-left: auto;
  background: #f58220;
  color: white;
  border: none;
  padding: 8px 20px;
  border-radius: 6px;
  font-weight: bold;
  cursor: pointer;
}

/* --- GRID LAYOUT --- */
.content-grid {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.kpi-row {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  gap: 12px;
}

.kpi-card {
  padding: 12px;
  border-radius: 8px;
  border: 1px solid #e0e0e0;
  box-shadow: 0 2px 4px rgba(0,0,0,0.02);
}

.kpi-header { font-size: 9px; font-weight: bold; color: #666; margin-bottom: 5px; text-transform: uppercase; }
.kpi-value { font-size: 18px; font-weight: 800; margin-bottom: 5px; }
.kpi-footer { display: flex; align-items: center; justify-content: space-between; }
.sub-val { font-size: 10px; color: #888; }
.perc-tag { 
  background: #e6f7ed; 
  color: #008a3e; 
  font-size: 10px; 
  font-weight: bold; 
  padding: 1px 6px; 
  border-radius: 4px;
}

.charts-row {
  display: grid;
  grid-template-columns: 1fr 280px;
  gap: 12px;
}

.pbi-card {
  background: white;
  border-radius: 8px;
  border: 1px solid #e0e0e0;
  padding: 15px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  font-weight: bold;
  color: #444;
  margin-bottom: 15px;
}

/* --- CHARTS STYLING --- */
.bar-chart-annual, .segment-viz {
  height: 150px;
  display: flex;
  align-items: flex-end;
  justify-content: space-around;
  padding-top: 20px;
}

.bar-item, .seg-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1;
  height: 100%;
  justify-content: flex-end;
}

.bar {
  width: 70%;
  background: #cbd5e1;
  border-radius: 2px 2px 0 0;
  transition: height 0.3s;
}
.bar.orange { background: #f58220; }
.bar.yellow { background: #eab308; width: 40px;}
.bar.blue { background: #3b82f6; width: 40px; }

.bar-val { font-size: 9px; font-weight: bold; margin-bottom: 4px; }
.bar-label { font-size: 10px; color: #999; margin-top: 5px; }

/* --- MAIN VIZ --- */
.main-viz { display: flex; flex-direction: column; }
.main-viz-header { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 10px; }
.title-area h3 { margin: 0; font-size: 15px; }
.title-area p { margin: 2px 0; font-size: 11px; color: #888; }

.main-filters { display: flex; gap: 10px; }
.year-selector { display: flex; background: #eee; padding: 2px; border-radius: 4px; }
.year-selector button { border: none; background: transparent; padding: 3px 10px; font-size: 11px; cursor: pointer; }
.year-selector button.active { background: #f58220; color: white; border-radius: 2px; }
.btn-export { border: 1px solid #ddd; background: white; padding: 4px 10px; font-size: 11px; border-radius: 4px; cursor: pointer;}

.legend-brands { display: flex; flex-wrap: wrap; gap: 15px; margin-bottom: 15px; }
.leg-brand { display: flex; align-items: center; gap: 6px; font-size: 11px; color: #666; font-weight: 500; }
.line-dot { width: 12px; height: 3px; border-radius: 2px; }

.chart-content-wrapper { display: flex; gap: 20px; flex: 1; }
.svg-container { flex: 1; min-height: 300px; }

.y-label, .x-label { font-size: 11px; fill: #999; font-weight: 600; }
.bar-text { font-size: 10px; fill: #444; font-weight: bold; }

/* --- SHARE COLUMN --- */
.share-column {
  width: 140px;
  background: #fcfcfc;
  border-left: 1px solid #eee;
  padding: 0 10px;
}
.share-header { font-size: 11px; font-weight: bold; margin-bottom: 15px; color: #3b82f6; }
.share-item { display: flex; align-items: center; margin-bottom: 14px; gap: 8px; }
.brand-logo-mini { width: 22px; height: 22px; object-fit: contain; }
.share-name { font-size: 10px; flex: 1; color: #666; font-weight: 600; }
.share-badge { 
  font-size: 10px; 
  font-weight: bold; 
  padding: 2px 6px; 
  border-radius: 4px; 
  min-width: 35px;
  text-align: center;
}

/* --- AI LEGEND FOOTER --- */
.ai-legend-card {
  background: #1a1a1a;
  color: white;
  padding: 15px 20px;
  border-radius: 10px;
  display: flex;
  gap: 15px;
  align-items: flex-start;
  margin-top: 10px;
}
.ai-icon {
  background: #333;
  color: #f58220;
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  font-size: 20px;
}
.ai-title { font-weight: bold; font-size: 14px; margin-bottom: 5px; display: flex; align-items: center; gap: 10px;}
.badge-ia { background: #332211; color: #f58220; font-size: 10px; padding: 2px 8px; border-radius: 4px; border: 1px solid #443322; }
.ai-content p { margin: 0; font-size: 12px; line-height: 1.5; color: #bbb; }

/* Responsividade Básica */
@media (max-width: 1200px) {
  .kpi-row { grid-template-columns: repeat(3, 1fr); }
  .charts-row { grid-template-columns: 1fr; }
}
</style>
