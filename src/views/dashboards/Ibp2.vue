<script setup>
import { ref } from 'vue';

// Dados simulados para as fatias das barras (proporções baseadas no print)
const stackValues = [
  { blue: 25, lightblue: 15, pink: 15, purple: 18, yellow: 15, teal: 12 }, // Jan
  { blue: 22, lightblue: 14, pink: 14, purple: 16, yellow: 14, teal: 20 }, // Fev
  { blue: 28, lightblue: 16, pink: 16, purple: 18, yellow: 12, teal: 10 }, // Mar
  { blue: 27, lightblue: 15, pink: 15, purple: 17, yellow: 16, teal: 10 }, // Abr
  { blue: 30, lightblue: 18, pink: 18, purple: 15, yellow: 10, teal: 9  }, // Mai
  { blue: 25, lightblue: 14, pink: 14, purple: 16, yellow: 15, teal: 16 }, // Jun
  { blue: 29, lightblue: 16, pink: 16, purple: 18, yellow: 14, teal: 7  }, // Jul
  { blue: 28, lightblue: 15, pink: 15, purple: 17, yellow: 15, teal: 10 }, // Ago
  { blue: 28, lightblue: 15, pink: 15, purple: 18, yellow: 14, teal: 10 }, // Set
  { blue: 30, lightblue: 17, pink: 17, purple: 19, yellow: 12, teal: 5  }, // Out
  { blue: 24, lightblue: 14, pink: 14, purple: 16, yellow: 14, teal: 18 }, // Nov
  { blue: 20, lightblue: 12, pink: 12, purple: 14, yellow: 14, teal: 28 }, // Dez
];

// Coordenadas Y para a linha preta (tendência)
const linePoints = "40,150 120,170 200,145 285,150 365,120 450,150 535,125 615,125 700,120 780,115 860,145 940,165";

const paletaPBI = [
  '#084594', // Azul Escuro (Base)
  '#2171b5', // Azul Médio
  '#f46d43', // Laranja/Coral
  '#6a3d9a', // Roxo
  '#df65b0', // Rosa/Magenta
  '#984ea3', // Violeta
  '#e6ab02', // Amarelo/Ouro
  '#d73027', // Vermelho
  '#006d2c', // Verde Escuro
  '#238b45', // Verde Médio
  '#41b6c4'  // Azul Turquesa (Topo)
];

const mesesLubrificantes = [
  { nome: 'janeiro',   total: 98,  fatias: [15, 2, 8, 2, 12, 10, 15, 5, 18, 8, 3] },
  { nome: 'fevereiro', total: 94,  fatias: [14, 2, 9, 2, 11, 9, 14, 4, 16, 10, 3] },
  { nome: 'março',     total: 100, fatias: [16, 2, 10, 3, 13, 11, 15, 5, 17, 6, 2] },
  { nome: 'abril',     total: 100, fatias: [16, 2, 10, 2, 12, 10, 16, 6, 18, 6, 2] },
  { nome: 'maio',      total: 107, fatias: [18, 2, 11, 3, 14, 12, 18, 5, 19, 3, 2] },
  { nome: 'junho',     total: 101, fatias: [15, 2, 10, 2, 12, 10, 15, 4, 17, 12, 2] },
  { nome: 'julho',     total: 109, fatias: [16, 2, 12, 3, 15, 13, 19, 5, 20, 2, 2] },
  { nome: 'agosto',    total: 110, fatias: [16, 2, 12, 2, 15, 13, 18, 4, 21, 5, 2] },
  { nome: 'setembro',  total: 110, fatias: [16, 2, 12, 2, 15, 13, 18, 6, 20, 4, 2] },
  { nome: 'outubro',   total: 112, fatias: [17, 2, 13, 3, 16, 14, 19, 6, 18, 2, 2] },
  { nome: 'novembro',  total: 98,  fatias: [14, 2, 11, 2, 13, 11, 16, 4, 15, 8, 2] },
  { nome: 'dezembro',  total: 89,  fatias: [12, 2, 10, 2, 11, 10, 14, 3, 13, 10, 2] },
];

// Função para gerar os pontos das linhas amarelas passando pelo CENTRO de cada fatia
const getLinhasPontosCentro = (indiceFatia) => {
  return mesesLubrificantes.map((m, i) => {
    const x = 40 + (i * 85); // Posição horizontal (centro da coluna)

    // 1. Soma todos os valores das fatias que estão ABAIXO da fatia atual
    const somaAteAnterior = m.fatias.slice(0, indiceFatia).reduce((a, b) => a + b, 0);
    
    // 2. Pega metade da altura da fatia ATUAL
    const metadeDestaFatia = m.fatias[indiceFatia] / 2;
    
    // 3. Soma (Anteriores + Metade da Atual) para achar o centro vertical
    const somaTotalFatias = m.fatias.reduce((a, b) => a + b, 0);
    const proporcaoCentro = (somaAteAnterior + metadeDestaFatia) / somaTotalFatias;
    
    // 4. Converte para a coordenada Y do SVG (escala de 250px baseado no total do mês)
    const y = 250 - (proporcaoCentro * (m.total * 1.8));
    
    return `${x},${y}`;
  }).join(' ');
};

// Cálculo automático da linha de tendência baseado nos totais
// (Isso gera os pontos X,Y para o SVG)
const pontosLinha = mesesLubrificantes.map((m, i) => {
  const x = 40 + (i * 85); // Espaçamento horizontal
  const y = 250 - (m.total * 1.8); // Inversão de altura para o SVG
  return `${x},${y}`;
}).join(' ');

const KpiIcon = `
<svg width="32" height="32" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
  <path d="M3 18H5V12H3V18ZM7 18H9V8H7V18ZM11 18H13V14H11V18ZM15 18H17V10H15V18ZM19 18H21V6H19V18Z" fill="#F58220" fill-opacity="0.3"/>
  <path d="M3 15L8 10L12 14L21 5M21 5H16M21 5V10" stroke="#F58220" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
</svg>`;
</script>

<template>
  <div class="pbi-dashboard">
    <!-- FILTROS -->
    <header class="pbi-header">
      <div class="f-group"><span>ANO:</span> <select><option>2025</option></select></div>
      <div class="f-group"><span>DO MÊS:</span> <select><option>Todos</option></select></div>
      <div class="f-group"><span>ATÉ O MÊS:</span> <select><option>Todos</option></select></div>
      <div class="f-group">
        <span>SEGMENTO:</span>
        <div class="pbi-tabs"><button>CONSUMIDOR</button><button class="active">REVENDEDOR</button></div>
      </div>
      <div class="f-group"><span>TIPO PRODUTO:</span> <select><option>Todos</option></select></div>
      <div class="f-group flex-grow-1"><span>SELECIONE A EMPRESA:</span> <select class="w-40"><option>TOTALENERGIES</option></select></div>
      <button class="btn-pbi-red">Limpar Filtros</button>
    </header>

    <div class="pbi-content">
      <!-- LINHA SUPERIOR -->
      <div class="top-row">
        <!-- KPIs -->
        <div class="kpi-grid">
          <div class="pbi-card kpi-card" v-for="n in 6" :key="n">
            <div class="kpi-inner">
              <div class="kpi-icon" v-html="KpiIcon"></div>
              <div class="kpi-text">
                <div class="kpi-label">{{ n <= 3 ? 'ANÁLISE MERCADO' : 'ANÁLISE TOTALENERGIES' }} - {{ n % 3 === 1 ? 'ANUAL' : n % 3 === 2 ? 'MENSAL' : 'PROJEÇÃO' }}</div>
                <div class="kpi-val">{{ n <= 3 ? '1.230.987 m³' : '16.617 m³' }}</div>
                <div class="kpi-sub">
                  <span>{{ n <= 3 ? '1.216.760 m³' : '15.826 m³' }}</span>
                  <span class="pbi-dot">●</span>
                  <span class="pbi-perc">{{ n <= 3 ? '1,17%' : '5,00%' }}</span>
                </div>
                <div class="kpi-footer-msg">%YoY {{ n % 3 === 1 ? 'Anual' : n % 3 === 2 ? 'Mensal' : 'Projeção' }}</div>
              </div>
              <div class="kpi-icon mirror" v-html="KpiIcon"></div>
            </div>
          </div>
        </div>

        <!-- VENDA ANUAL -->
        <!-- VENDA ANUAL CORRIGIDO -->
        <div class="pbi-card side-chart">
        <div class="chart-header">VENDA ANUAL</div>
        <div class="venda-anual-bars">
            <div v-for="(v, i) in [
            {y:'2017', v:'21.435', n:21435}, {y:'2018', v:'', n:21435}, 
            {y:'2019', v:'19.275', n:19275}, {y:'2020', v:'16.946', n:16946}, 
            {y:'2021', v:'22.097', n:22097}, {y:'2022', v:'18.245', n:18245}, 
            {y:'2023', v:'15.133', n:15133}, {y:'2024', v:'', n:14000},
            {y:'2025', v:'16.617', n:16617}, {y:'2026', v:'2.944', n:4000}
            ]" :key="i" class="va-col">
            <!-- Valor no topo -->
            <span class="va-txt">{{ v.v }}</span>
            <!-- Barra Laranja (com lógica para a última ser clara) -->
            <div class="va-bar" 
                :class="{ 'last-bar': i === 9 }"
                :style="{ height: (v.n / 25000 * 100) + '%' }">
            </div>
            <!-- Ano no rodapé -->
            <span class="va-year">{{ v.y }}</span>
            </div>
        </div>
        </div>

        <!-- SEGMENTO -->
        <div class="pbi-card side-chart segment">
          <div class="chart-header text-center">Vendas de Lubrificantes por Segmento</div>
          <div class="seg-viz">
            <div class="seg-bar-group">
              <span class="va-txt">15.239</span>
              <div class="va-bar main" style="height: 120px; width: 45px;"></div>
              <span class="va-year font-bold">REVENDEDOR</span>
            </div>
            <div class="seg-bar-group">
              <span class="va-txt">1.378</span>
              <div class="va-bar light" style="height: 20px; width: 45px;"></div>
              <span class="va-year font-bold">CONSUMIDOR</span>
            </div>
          </div>
        </div>
      </div>

      <!-- GRÁFICO INFERIOR ADAPTADO -->
      <div class="pbi-card main-viz-card">
        <div class="chart-header px-4 pt-3">Vendas de Lubrificantes por mês</div>
        
        <div class="main-viz-wrapper">
            <!-- Camada das Linhas de Tendência Amarelas (Passando pelo Centro) -->
            <svg class="svg-line-layer" viewBox="0 0 1000 300" preserveAspectRatio="none">
                <polyline v-for="n in 11" :key="n"
                    :points="getLinhasPontosCentro(n-1)" 
                    fill="none" 
                    stroke="#ffff00" 
                    stroke-width="2.5" 
                    style="filter: drop-shadow(0px 1px 2px rgba(0,0,0,0.4))" 
                />
            </svg>

            <!-- Colunas Empilhadas -->
            <div class="stacked-bars-container">
                <div v-for="m in mesesLubrificantes" :key="m.nome" class="month-stack-col">
                    <span class="stack-top-label">{{ m.total }} Mil</span>
                    
                    <div class="stack-body" :style="{ height: (m.total / 112 * 90) + '%' }">
                        <!-- Gerar as fatias dinamicamente com a paleta -->
                        <div v-for="(fatia, fIdx) in m.fatias" :key="fIdx"
                             :style="{ flex: fatia, backgroundColor: paletaPBI[fIdx] }">
                        </div>
                    </div>
                    
                    <span class="month-name">{{ m.nome }}</span>
                </div>
            </div>
        </div>

        <!-- Legenda (Opcional, conforme o print) -->
        <div class="pbi-legend">
           <span class="leg-item"><i style="background:#0070c0"></i> YPF</span>
           <span class="leg-item"><i style="background:#1a237e"></i> VIBRA</span>
           <span class="leg-item"><i style="background:#c65911"></i> VALVOLINE</span>
           <span class="leg-item"><i style="background:#7030a0"></i> TOTALENERGIES</span>
           <span class="leg-item"><i style="background:#ffc000"></i> MOOVE</span>
           <span class="leg-item"><i style="background:#264d59"></i> ICONIC</span>
        </div>
      </div>

    </div>
  </div>
</template>

<style scoped>
.pbi-dashboard {
  background-color: #f3f3f3; height: 100vh; display: flex; flex-direction: column;
  font-family: 'Segoe UI', Arial, sans-serif; color: #333;
}

/* Filtros */
.pbi-header {
  background: white; padding: 10px 15px; display: flex; align-items: center; gap: 15px;
  font-size: 11px; font-weight: bold; border-bottom: 1px solid #ddd;
}
.f-group { display: flex; align-items: center; gap: 6px; }
.f-group select { border: 1px solid #ccc; padding: 2px 5px; border-radius: 2px; }
.pbi-tabs { border: 1px solid #ccc; border-radius: 2px; display: flex; }
.pbi-tabs button { border: none; background: white; padding: 2px 10px; font-size: 10px; cursor: pointer; color: #777; }
.pbi-tabs button.active { background: #eee; color: #000; font-weight: bold; }
.btn-pbi-red { background: #ff4d4d; color: white; border: none; padding: 6px 15px; border-radius: 6px; font-weight: bold; margin-left: auto; cursor: pointer; }

/* Grid Proporcional */
.pbi-content { flex: 1; padding: 15px; display: flex; flex-direction: column; gap: 15px; }
.top-row { display: grid; grid-template-columns: 3fr 1fr 1fr; gap: 15px; height: 280px; }
.kpi-grid { display: grid; grid-template-columns: repeat(3, 1fr); grid-template-rows: repeat(2, 1fr); gap: 12px; }

.pbi-card { background: white; border-radius: 15px; border: 1px solid #e5e5e5; box-shadow: 0 2px 5px rgba(0,0,0,0.03); }

/* KPI Styling */
.kpi-card { padding: 12px; }
.kpi-inner { display: flex; justify-content: space-between; align-items: center; height: 100%; }
.kpi-text { flex: 1; margin: 0 10px; }
.kpi-label { font-size: 10px; font-weight: 800; color: #222; }
.kpi-val { font-size: 24px; font-weight: 800; color: #000; line-height: 1.1; margin: 4px 0; }
.kpi-sub { font-size: 11px; display: flex; align-items: center; gap: 4px; font-weight: 600; color: #666; }
.pbi-dot { color: #228b22; font-size: 14px; }
.kpi-footer-msg { font-size: 10px; color: #aaa; margin-top: 3px; }
.kpi-icon.mirror { opacity: 0.15; transform: scaleX(-1); }

/* Charts Laterais */
.side-chart { padding: 15px; display: flex; flex-direction: column; }
.chart-header { font-size: 12px; font-weight: bold; color: #333; margin-bottom: 15px; }
.venda-anual-bars {
  flex: 1;
  display: flex;
  align-items: flex-end; /* Alinha as barras na base */
  justify-content: space-between;
  padding-top: 20px; /* Espaço para o texto não cortar */
  height: 180px; /* Altura fixa para garantir que as barras apareçam */
}

.va-col {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  height: 100%;
  justify-content: flex-end;
}

.va-bar {
  background-color: #f58220; /* Laranja Power BI */
  width: 70%; /* Largura da barra */
  border-radius: 2px 2px 0 0;
  min-height: 2px; /* Garante visibilidade mesmo se valor for baixo */
  transition: height 0.3s ease;
}

.va-bar.last-bar {
  background-color: #fcd5b5; /* Laranja claro para o último ano (2026) */
}

.va-txt {
  font-size: 8px;
  font-weight: bold;
  color: #666;
  margin-bottom: 4px;
  white-space: nowrap;
}

.va-year {
  font-size: 9px;
  color: #999;
  margin-top: 6px;
}
.seg-viz { flex: 1; display: flex; justify-content: space-around; align-items: flex-end; padding-bottom: 10px; }
.seg-bar-group { display: flex; flex-direction: column; align-items: center; }

.main-viz-card {
  flex: none; 
  display: flex;
  flex-direction: column;
  height: 500px; /* Aumentado para acomodar título e legenda */
  overflow: hidden; 
}

.main-viz-wrapper {
  flex: 1;
  position: relative;
  width: 100%;
  margin-top: 20px;
}

/* Container das colunas */
.stacked-bars-container {
  display: flex;
  width: 100%;
  height: 100%; /* Ocupa toda a altura do wrapper */
  justify-content: space-between;
  align-items: flex-end;
  z-index: 1;
}

/* Coluna individual de cada mês */
.month-stack-col {
  flex: 1;
  height: 100%; /* Faz a coluna ocupar todo o espaço vertical para o cálculo de % funcionar */
  display: flex;
  flex-direction: column;
  justify-content: flex-end; /* Alinha o conteúdo (label, barra, mês) na base */
  align-items: center;
}

/* A BARRA: Aqui corrigimos a percepção de altura */
.stack-body {
  width: 58px; /* Largura um pouco maior conforme o print */
  display: flex;
  flex-direction: column-reverse;
  border-radius: 0; 
  overflow: hidden;
  margin-bottom: 5px; 
}

/* Labels e Textos */
.stack-top-label {
  font-size: 10px;
  font-weight: bold;
  color: #777;
  margin-bottom: 8px;
}

/* Legenda no rodapé */
.pbi-legend {
  display: flex;
  justify-content: center;
  gap: 12px;
  padding: 10px;
  flex-wrap: wrap;
}

.leg-item {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 9px;
  font-weight: bold;
  color: #666;
  text-transform: uppercase;
}

.leg-item i {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

.month-name {
  font-size: 10px;
  color: #666;
  font-weight: 600;
  padding-top: 5px;
}

/* Linha de Tendência - Deve cobrir a área das barras */
.svg-line-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 90%; 
  pointer-events: none;
  z-index: 2;
}

/* Paleta de Cores Exata */
.s-blue   { background: #1a237e; }
.s-pink   { background: #ad1457; }
.s-purple { background: #6a1b9a; }
.s-yellow { background: #f9a825; }
.s-teal   { background: #00695c; }
.s-green  { background: #2e7d32; }

.font-bold { font-weight: 800; }

</style>