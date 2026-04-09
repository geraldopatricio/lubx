<script setup>
import { ref } from 'vue';

const marcas = [
  { nome: 'TOTALENERGIES', cor: '#ff0000', share: '32,1%', logo: 'totalenergies' },
  { nome: 'YPF', cor: '#0070c0', share: '24,8%', logo: 'ypf' },
  { nome: 'VIBRA', cor: '#004415', share: '15,2%', logo: 'vibra' },
  { nome: 'PETRONAS', cor: '#00a19c', share: '11,4%', logo: 'petronas' },
  { nome: 'CASTROL', cor: '#f00023', share: '8,9%', logo: 'castrol' },
  { nome: 'VALVOLINE', cor: '#d1d5d8', share: '5,6%', logo: 'valvoline' },
  { nome: 'RAIZEN', cor: '#990093', share: '2,0%', logo: 'raizen' },
  { nome: 'MOOVE', cor: '#85bd00', share: '2,0%', logo: 'moove' },
  { nome: 'LWART', cor: '#006cb7', share: '2,0%', logo: 'lwart' },
  { nome: 'ICONIC', cor: '#ff511f', share: '2,0%', logo: 'iconic' }
];

const kpis = [
  { label: 'ANÁLISE MERCADO - ANUAL', val: '1.230.987 m³', sub: '1.216.760 m³', perc: '+1,17%', bg: '#fff' },
  { label: 'ANÁLISE MERCADO - MENSAL', val: '1.230.987 m³', sub: '1.216.760 m³', perc: '+1,17%', bg: '#fff' },
  { label: 'ANÁLISE MERCADO - PROJEÇÃO', val: '1.230.987 m³', sub: '1.233.992 m³', perc: '+1,17%', bg: '#fff' },
  { label: 'TotalEnergies - ANUAL', val: '16.617 m³', sub: '15.826 m³', perc: '+5,00%', bg: '#fff6ed' },
  { label: 'TotalEnergies - MENSAL', val: '16.617 m³', sub: '15.826 m³', perc: '+5,00%', bg: '#eef6ff' },
  { label: 'TotalEnergies - PROJEÇÃO', val: '16.617 m³', sub: '15.826 m³', perc: '+5,00%', bg: '#f1f1f1' },
];

const vendaAnualBars = [
  { ano: '2017', val: '21.154', h: 60 }, { ano: '2018', val: '24.500', h: 70 },
  { ano: '2019', val: '24.100', h: 68 }, { ano: '2020', val: '15.004', h: 42 },
  { ano: '2021', val: '28.984', h: 82 }, { ano: '2022', val: '30.354', h: 86 },
  { ano: '2023', val: '31.200', h: 88 }, { ano: '2024', val: '28.700', h: 81 },
  { ano: '2025', val: '35.445', h: 100, active: true }
];

const meses = ['Jan', 'Fev', 'Mar', 'Abr', 'Mai', 'Jun', 'Jul', 'Ago', 'Set', 'Out', 'Nov', 'Dez'];
const mainChartData = [
  { val: '400k', bar: 180, line: 150 }, { val: '380k', bar: 170, line: 160 },
  { val: '405k', bar: 185, line: 145 }, { val: '399k', bar: 180, line: 155 },
  { val: '410k', bar: 190, line: 140 }, { val: '420k', bar: 200, line: 145 },
  { val: '410k', bar: 190, line: 140 }, { val: '410k', bar: 190, line: 140 },
  { val: '410k', bar: 190, line: 142 }, { val: '460k', bar: 230, line: 120, active: true },
  { val: '415k', bar: 195, line: 140 }, { val: '375k', bar: 165, line: 155 }
];
</script>

<template>
  <div class="pbi-layout">
    <!-- FILTROS -->
    <header class="pbi-filters">
      <div class="filter-left">
        <div class="pbi-select">TotalEnergies <span class="arrow">▼</span></div>
        <div class="pbi-tabs"><button>CONSUMIDOR</button><button class="active">REVENDEDOR</button></div>
        <div class="pbi-date">📅 02/17 – 31/12/2025 <span class="arrow">▼</span></div>
      </div>
      <button class="btn-clear">Limpar Filtros</button>
    </header>

    <div class="pbi-content">
      <!-- SEÇÃO SUPERIOR: KPIs + GRÁFICOS LADO A LADO -->
      <section class="top-section">
        <!-- Grid de KPIs: 3 colunas x 2 linhas -->
        <div class="kpi-grid">
          <div v-for="(k, i) in kpis" :key="i" class="card kpi-card" :style="{ background: k.bg }">
            <div class="kpi-label">{{ k.label }}</div>
            <div class="kpi-val">{{ k.val }}</div>
            <div class="kpi-footer">
              <span class="sub">{{ k.sub }}</span>
              <span class="pill">{{ k.perc }}</span>
            </div>
          </div>
        </div>

        <!-- Venda Anual -->
        <div class="card chart-annual">
          <div class="card-title">Venda Anual <span class="ico">⭳</span></div>
          <div class="bars-container">
            <div v-for="b in vendaAnualBars" :key="b.ano" class="bar-col">
              <span class="bar-txt">{{ b.val }}</span>
              <div class="bar" :class="{ orange: b.active }" :style="{ height: b.h + '%' }"></div>
              <span class="bar-label">{{ b.ano }}</span>
            </div>
          </div>
        </div>

        <!-- Vendas por Segmento -->
        <div class="card chart-segment">
          <div class="card-title">Vendas por Segmento <span class="ico">⭳</span></div>
          <div class="segment-container">
            <div class="seg-col">
              <span class="bar-txt">35.445</span>
              <div class="seg-bar yellow" style="height: 85%"></div>
              <span class="bar-label bold">Revendedor</span>
            </div>
            <div class="seg-col">
              <span class="bar-txt">35.445</span>
              <div class="seg-bar blue" style="height: 60%"></div>
              <span class="bar-label bold">Consumidor</span>
            </div>
          </div>
        </div>
      </section>

      <!-- GRÁFICO PRINCIPAL -->
      <section class="card main-viz">
        <div class="viz-header">
          <div class="viz-info">
            <h3>Vendas de Lubrificantes por Mês</h3>
            <p>Barras = Média de Mercado (contexto) • Linhas = Volume por Marca • Coluna % = Share de Venda</p>
          </div>
          <div class="viz-actions">
            <div class="tabs"><button>2023</button><button>2024</button><button class="active">2025</button></div>
            <button class="btn-exp">⭳ Exportar</button>
          </div>
        </div>

        <div class="viz-legend">
          <div v-for="m in marcas" :key="m.nome" class="leg-item">
            <span class="dot" :style="{ background: m.cor }"></span> {{ m.nome }}
          </div>
        </div>

        <div class="viz-body">
          <div class="svg-area">
            <svg viewBox="0 0 1000 320" preserveAspectRatio="none">
              <line v-for="n in 5" :x1="0" :y1="280 - (n*55)" x2="1000" :y2="280 - (n*55)" stroke="#f0f0f0" />
              <text v-for="n in 6" :x="5" :y="285 - (n-1)*55" class="svg-txt">{{ (n-1)*100 }}M</text>
              <g v-for="(d, i) in mainChartData" :key="i">
                <rect :x="i*82 + 50" :y="280 - d.bar" width="42" :height="d.bar" :fill="d.active ? '#ffb366' : '#e9ecef'" rx="2" />
                <text :x="i*82 + 71" :y="280 - d.bar - 10" text-anchor="middle" class="bar-val">R$ {{ d.val }}</text>
              </g>
              <polyline :points="mainChartData.map((d, i) => `${i*82 + 71},${d.line}`).join(' ')" fill="none" stroke="#f58220" stroke-width="4" />
              <circle v-for="(d, i) in mainChartData" :cx="i*82 + 71" :cy="d.line" r="5" fill="#f58220" />
              <text v-for="(m, i) in meses" :x="i*82 + 71" y="310" text-anchor="middle" class="svg-txt">{{ m }}</text>
            </svg>
          </div>

          <div class="sidebar">
            <div class="side-title">% Share</div>
            <div v-for="m in marcas" :key="m.nome" class="side-item">
              <img :src="`/logos/lubrificantes/${m.logo}.png`" class="mini-logo" />
              <span class="name">{{ m.nome }}</span>
              <span class="badge" :class="{ orange: m.nome === 'TotalEnergies' }">{{ m.share }}</span>
            </div>
          </div>
        </div>
      </section>

      <!-- LEGENDA IA -->
      <footer class="ia-legend">
        <div class="ia-ico">✦</div>
        <div class="ia-content">
          <div class="ia-title">Dashboard Legenda por IA <span class="ia-tag">LubAssist IA</span></div>
          <p>Este dashboard apresenta análise de market share (IBP). Os 6 cards superiores comparam o mercado total com a TotalEnergies (em laranja) nas visões anual, mensal e projetada. O gráfico central sobrepõe barras cinzas (média de mercado como contexto) com linhas por marca — laranja para TotalEnergies, tons de cinza para concorrentes. A coluna direita exibe o % de share de cada marca. Pico sazonal observado entre Maio e Outubro.</p>
        </div>
      </footer>
    </div>
  </div>
</template>

<style scoped>
/* ESTRUTURA RÍGIDA PARA MANTER POSIÇÕES EM QUALQUER TELA */
.pbi-layout {
  background-color: #f3f3f3;
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  padding: 15px;
  box-sizing: border-box;
  overflow: auto;
}

.pbi-content {
  min-width: 1300px; /* Impede que os cards se quebrem em telas pequenas */
  display: flex;
  flex-direction: column;
  gap: 15px;
  flex: 1;
}

/* FILTROS */
.pbi-filters {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: white;
  padding: 10px 20px;
  border-radius: 10px;
  margin-bottom: 15px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.05);
}
.filter-left { display: flex; gap: 15px; align-items: center; }
.pbi-select { border: 1px solid #ddd; padding: 6px 15px; border-radius: 5px; color: #f58220; font-weight: bold; min-width: 160px; display: flex; justify-content: space-between; }
.pbi-tabs { display: flex; background: #eee; padding: 3px; border-radius: 8px; }
.pbi-tabs button { border: none; background: transparent; padding: 5px 15px; font-size: 11px; font-weight: bold; color: #777; cursor: pointer; }
.pbi-tabs button.active { background: #f58220; color: white; border-radius: 6px; }
.pbi-date { border: 1px solid #ddd; padding: 6px 15px; border-radius: 5px; font-size: 12px; }
.btn-clear { background: #f58220; color: white; border: none; padding: 10px 20px; border-radius: 8px; font-weight: bold; cursor: pointer; }

/* SEÇÃO SUPERIOR: KPIs + GRÁFICOS */
.top-section {
  display: grid;
  grid-template-columns: 3fr 1.5fr 1fr; /* Proporção das 3 áreas principais */
  gap: 15px;
  height: 260px;
}

/* KPI GRID: 3 colunas x 2 linhas */
.kpi-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: repeat(2, 1fr);
  gap: 10px;
}
.card { background: white; border: 1px solid #e0e0e0; border-radius: 10px; padding: 12px; }
.kpi-card { display: flex; flex-direction: column; justify-content: space-between; }
.kpi-label { font-size: 9px; font-weight: bold; color: #666; text-transform: uppercase; }
.kpi-val { font-size: 20px; font-weight: 800; }
.kpi-footer { display: flex; justify-content: space-between; align-items: center; }
.sub { font-size: 10px; color: #999; }
.pill { background: #e6f7ed; color: #008a3e; font-size: 10px; font-weight: bold; padding: 2px 8px; border-radius: 5px; }

/* GRÁFICOS SUPERIORES */
.chart-annual, .chart-segment { display: flex; flex-direction: column; }
.card-title { font-size: 12px; font-weight: bold; margin-bottom: 15px; display: flex; justify-content: space-between; }
.bars-container, .segment-container { flex: 1; display: flex; align-items: flex-end; justify-content: space-between; padding-bottom: 5px; }
.bar-col, .seg-col { flex: 1; display: flex; flex-direction: column; align-items: center; height: 100%; justify-content: flex-end; }
.bar { width: 60%; background: #bdc2c6; border-radius: 3px 3px 0 0; }
.bar.orange { background: #f58220; }
.seg-bar { width: 45px; border-radius: 3px; }
.seg-bar.yellow { background: #eab308; }
.seg-bar.blue { background: #3b82f6; }
.bar-txt { font-size: 9px; font-weight: bold; margin-bottom: 5px; }
.bar-label { font-size: 10px; color: #999; margin-top: 8px; }
.bold { font-weight: bold; color: #444; }

/* GRÁFICO PRINCIPAL */
.main-viz { flex: 1; display: flex; flex-direction: column; min-height: 400px; }
.viz-header { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 10px; }
.viz-info h3 { margin: 0; font-size: 18px; }
.viz-info p { margin: 3px 0; font-size: 11px; color: #888; }
.tabs { display: flex; background: #eee; padding: 3px; border-radius: 8px; }
.tabs button { border: none; background: transparent; padding: 5px 15px; font-size: 11px; cursor: pointer; }
.tabs button.active { background: #f58220; color: white; border-radius: 6px; }
.btn-exp { border: 1px solid #ddd; background: white; padding: 6px 15px; font-size: 12px; border-radius: 6px; margin-left: 10px; cursor: pointer; }

.viz-legend { display: flex; gap: 20px; margin: 15px 0; }
.leg-item { display: flex; align-items: center; gap: 6px; font-size: 11px; color: #666; }
.dot { width: 14px; height: 3px; border-radius: 2px; }

.viz-body { display: flex; flex: 1; gap: 30px; }
.svg-area { flex: 1; position: relative; }
.svg-txt { font-size: 11px; fill: #aaa; font-weight: 600; }
.bar-val { font-size: 10px; font-weight: bold; fill: #444; }

/* SIDEBAR */
.sidebar { width: 180px; border-left: 1px solid #f0f0f0; padding-left: 20px; }
.side-title { font-size: 12px; font-weight: bold; color: #007bff; margin-bottom: 15px; }
.side-item { display: flex; align-items: center; gap: 10px; margin-bottom: 12px; }
.mini-logo { width: 22px; height: 22px; object-fit: contain; }
.name { font-size: 11px; flex: 1; color: #555; font-weight: 600; }
.badge { font-size: 10px; font-weight: bold; background: #f2f2f2; padding: 3px 8px; border-radius: 5px; min-width: 40px; text-align: center; }
.badge.orange { background: #f58220; color: white; }

/* IA FOOTER */
.ia-legend { background: #212529; color: white; padding: 20px; border-radius: 12px; display: flex; gap: 20px; margin-top: auto; }
.ia-ico { font-size: 24px; color: #f58220; }
.ia-title { font-weight: bold; font-size: 15px; margin-bottom: 6px; display: flex; align-items: center; gap: 10px; }
.ia-tag { background: #3d3429; color: #f58220; font-size: 10px; padding: 3px 10px; border-radius: 5px; border: 1px solid #5a4b39; }
.ia-content p { margin: 0; font-size: 12px; line-height: 1.6; color: #adb5bd; }
</style>
