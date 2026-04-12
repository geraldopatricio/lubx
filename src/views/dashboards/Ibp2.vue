<script setup>
import { ref, reactive, onMounted, watch, computed } from 'vue';

// const API_BASE = 'http://localhost:3000/bi/ipb';
const API_BASE = 'https://lubx-api.lubconsulta.com.br/bi/ipb';

const loading = ref(true);
const dashboardData = ref(null);

const state = reactive({
  empresa: 'TOTALENERGIES',
  ano: '2025',
  mesDe: 'Jan',
  mesAte: 'Dez',
  segmento: 'REVENDEDOR'
});

// Configuração de cores (Fidelidade visual)
const marcasConfig = {
  'TOTALENERGIES': { cor: '#f58220', logo: 'totalenergies' },
  'YPF': { cor: '#00529b', logo: 'ypf' },
  'VIBRA': { cor: '#004415', logo: 'vibra' },
  'PETRONAS': { cor: '#00A79D', logo: 'petronas' },
  'CASTROL': { cor: '#E10600', logo: 'castrol' },
  'VALVOLINE': { cor: '#2FA4E7', logo: 'valvoline' },
  'RAIZEN LUBRIFICANTES': { cor: '#9B1BB3', logo: 'raizen' },
  'MOOVE': { cor: '#00A3E0', logo: 'moove' },
  'LWART': { cor: '#000000', logo: 'lwart' },
  'ICONIC': { cor: '#F56600', logo: 'iconic' },
  'CHEVRON': { cor: '#005da4', logo: 'chevron' },
  'IPIRANGA': { cor: '#ffcb05', logo: 'ipiranga' }
};

async function fetchDashboard() {
  loading.value = true;
  try {
    const params = new URLSearchParams(state).toString();
    const response = await fetch(`${API_BASE}/dashboard?${params}`);
    const json = await response.json();
    if (json.status === 'ok') dashboardData.value = json.data;
  } catch (error) {
    console.error("Erro:", error);
  } finally {
    loading.value = false;
  }
}

// Lógica de Escalonamento para Múltiplas Linhas
const chartMetrics = computed(() => {
  if (!dashboardData.value?.graficos?.vendasPorMes) return null;
  const series = dashboardData.value.graficos.vendasPorMes.series;
  const meses = dashboardData.value.graficos.vendasPorMes.meses;
  
  // Volume total para as barras de fundo
  const marketVolumes = meses.map((_, i) => 
    series.reduce((sum, s) => sum + s.valores[i], 0)
  );

  const maxVal = Math.max(...marketVolumes) * 1.2;

  // Mapeia todas as séries (linhas)
  const allLines = series.map(s => ({
    empresa: s.empresa,
    cor: marcasConfig[s.empresa]?.cor || '#ccc',
    destaque: s.empresa === state.empresa,
    points: s.valores.map(v => (v / maxVal) * 220)
  }));

  return {
    bars: marketVolumes.map(v => (v / maxVal) * 220),
    allLines,
    labels: marketVolumes.map(v => `R$ ${Math.round(v/1000)}k`)
  };
});

onMounted(fetchDashboard);
watch(state, fetchDashboard);

const formatM3 = (v) => v.toLocaleString('pt-BR') + ' m³';
</script>

<template>
  <div class="pbi-layout" v-if="dashboardData">
    <!-- FILTROS AJUSTADOS PARA HORIZONTAL -->
    <header class="pbi-filters">
      <div class="filter-left">
        <div class="pbi-select">
          <select v-model="state.empresa">
            <option v-for="e in dashboardData.filtros.empresas" :key="e" :value="e">{{ e }}</option>
          </select>
        </div>
        <div class="pbi-tabs">
          <button v-for="s in dashboardData.filtros.segmentos" :key="s.id" 
                  :class="{ active: state.segmento === s.id }" @click="state.segmento = s.id">
            {{ s.label.toUpperCase() }}
          </button>
        </div>
        <div class="pbi-date">📅 {{ state.mesDe }} - {{ state.mesAte }} / {{ state.ano }}</div>
      </div>
      <button class="btn-clear" @click="state.ano = '2025'">Limpar Filtros</button>
    </header>

    <div class="pbi-content" :class="{ 'loading-opacity': loading }">
      <!-- KPIs -->
      <section class="top-section">
        <div class="kpi-grid">
          <div v-for="(k, i) in dashboardData.cards" :key="i" class="card kpi-card" 
               :style="{ background: k.id.includes('empresa') ? '#fff6ed' : '#fff' }">
            <div class="kpi-label">{{ k.titulo }}</div>
            <div class="kpi-val">{{ formatM3(k.valor) }}</div>
            <div class="kpi-footer">
              <span class="sub">{{ formatM3(k.comparativo) }}</span>
              <span class="pill" :class="{ neg: k.variacaoPercentual < 0 }">{{ k.variacaoPercentual }}% YoY</span>
            </div>
          </div>
        </div>

        <div class="card chart-annual">
          <div class="card-title">Venda Anual</div>
          <div class="bars-container">
            <div v-for="b in dashboardData.graficos.vendaAnual.items" :key="b.ano" class="bar-col">
              <span class="bar-txt">{{ Math.round(b.valor/1000) }}k</span>
              <div class="bar" :class="{ orange: b.ano == state.ano }" 
                   :style="{ height: (b.valor / Math.max(...dashboardData.graficos.vendaAnual.items.map(x=>x.valor)) * 100) + '%' }"></div>
              <span class="bar-label">{{ b.ano }}</span>
            </div>
          </div>
        </div>

        <div class="card chart-segment">
          <div class="card-title">Vendas por Segmento</div>
          <div class="segment-container">
            <div v-for="seg in dashboardData.graficos.segmento.items" :key="seg.id" class="seg-col">
              <span class="bar-txt">{{ Math.round(seg.valor/1000) }}k</span>
              <div class="seg-bar" :style="{ background: seg.cor, height: '70%' }"></div>
              <span class="bar-label bold">{{ seg.label }}</span>
            </div>
          </div>
        </div>
      </section>

      <!-- GRÁFICO PRINCIPAL COM MULTI-LINHAS -->
      <section class="card main-viz">
        <div class="viz-header">
          <div class="viz-info">
            <h3>Vendas de Lubrificantes por Mês</h3>
            <p>Barras = Volume Total Mercado • Linhas = Evolução por Marca</p>
          </div>
          <div class="viz-actions">
            <!-- BOTÕES DE ANO ESTILIZADOS -->
            <div class="tabs-anos">
              <button v-for="ano in dashboardData.filtros.anos" :key="ano" 
                      :class="{ active: state.ano === ano }" @click="state.ano = ano">{{ ano }}</button>
            </div>
          </div>
        </div>

        <!-- LEGENDA -->
        <div class="viz-legend">
          <div v-for="(cfg, nome) in marcasConfig" :key="nome" class="leg-item" @click="state.empresa = nome">
            <span class="dot" :style="{ background: cfg.cor }"></span> 
            <span :class="{ 'bold-active': state.empresa === nome }">{{ nome }}</span>
          </div>
        </div>

        <div class="viz-body">
          <div class="svg-area">
            <svg viewBox="0 0 1000 320" preserveAspectRatio="none">
              <!-- Grid -->
              <line v-for="n in 5" :key="n" x1="50" :y1="280 - (n*50)" x2="950" :y2="280 - (n*50)" stroke="#f0f0f0" />
              
              <!-- Barras -->
              <g v-for="(h, i) in chartMetrics?.bars" :key="'b'+i">
                <rect :x="i*75 + 60" :y="280 - h" width="40" :height="h" fill="#e9ecef" rx="2" />
                <text :x="i*75 + 80" :y="280 - h - 10" text-anchor="middle" class="bar-val">{{ chartMetrics.labels[i] }}</text>
              </g>

              <!-- MÚLTIPLAS LINHAS -->
              <g v-for="line in chartMetrics?.allLines" :key="line.empresa">
                <polyline 
                  :points="line.points.map((h, i) => `${i*75 + 80},${280 - h}`).join(' ')" 
                  fill="none" 
                  :stroke="line.cor" 
                  :stroke-width="line.destaque ? 4 : 1.5"
                  :stroke-opacity="line.destaque ? 1 : 0.4"
                  stroke-linejoin="round"
                />
                <g v-if="line.destaque">
                  <circle v-for="(h, i) in line.points" :key="'c'+i"
                          :cx="i*75 + 80" :cy="280 - h" r="4" :fill="line.cor" />
                </g>
              </g>

              <text v-for="(m, i) in dashboardData.graficos.vendasPorMes.meses" :key="m" 
                    :x="i*75 + 80" y="310" text-anchor="middle" class="svg-txt">{{ m }}</text>
            </svg>
          </div>

          <!-- SIDEBAR % SHARE COM LOGOS AJUSTADAS -->
          <div class="sidebar">
            <div class="side-title">% Share Mercado</div>
            <div v-for="m in dashboardData.marcas" :key="m.nome" class="side-item" @click="state.empresa = m.nome">
              <img :src="`/logos/lubrificantes/${marcasConfig[m.nome]?.logo || 'generic'}.png`" class="brand-logo-img" />
              <span class="name">{{ m.nome }}</span>
              <span class="badge" :class="{ orange: m.selecionada }">24,8%</span>
            </div>
          </div>
        </div>
      </section>

      <footer class="ia-legend">
        <div class="ia-ico">✦</div>
        <div class="ia-content">
          <div class="ia-title">Dashboard Legenda por IA <span class="ia-tag">LubAssist IA</span></div>
          <p>Visão de <b>{{ state.ano }}</b>: Comparativo multi-marcas. A linha em destaque representa <b>{{ state.empresa }}</b>.</p>
        </div>
      </footer>
    </div>
  </div>
</template>

<style scoped>
.pbi-layout { background: #f3f3f3; min-height: 100vh; padding: 15px; font-family: 'Segoe UI', sans-serif; }
.pbi-content { display: flex; flex-direction: column; gap: 15px; min-width: 1200px; transition: opacity 0.3s; }
.loading-opacity { opacity: 0.6; }

/* AJUSTE FILTROS LADO A LADO */
.pbi-filters { display: flex; justify-content: space-between; background: white; padding: 10px 20px; border-radius: 8px; align-items: center; margin-bottom: 10px; }
.filter-left { display: flex; flex-direction: row; align-items: center; gap: 20px; } /* Garante o alinhamento horizontal */

.pbi-tabs { display: flex; background: #eee; padding: 3px; border-radius: 6px; }
.pbi-tabs button { border: none; background: transparent; padding: 5px 12px; font-size: 11px; cursor: pointer; font-weight: bold; color: #666; }
.pbi-tabs button.active { background: #f58220; color: white; border-radius: 4px; }
.pbi-date { white-space: nowrap; font-size: 13px; color: #444; }

/* BOTÕES DE ANO - ESTILO CLEAN */
.tabs-anos { display: flex; gap: 6px; }
.tabs-anos button { border: 1px solid #e0e0e0; background: #fff; padding: 5px 14px; font-size: 12px; cursor: pointer; border-radius: 4px; color: #333; transition: 0.2s; }
.tabs-anos button.active { background: #f58220; color: white; border-color: #f58220; font-weight: bold; }

/* BOTÃO LIMPAR FILTROS */
.btn-clear { background: white; border: 2px solid #333; color: #333; padding: 8px 18px; border-radius: 4px; font-weight: bold; cursor: pointer; font-size: 12px; }

/* LOGOS SIDEBAR */
.brand-logo-img { width: 30px; height: 30px; object-fit: contain; border-radius: 6px; } /* Aumentado ~40% e sem bordas */

.top-section { display: grid; grid-template-columns: 3fr 1.5fr 1fr; gap: 15px; height: 260px; }
.kpi-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; }
.card { background: white; border-radius: 10px; border: 1px solid #e0e0e0; padding: 12px; display: flex; flex-direction: column; }
.kpi-val { font-size: 20px; font-weight: 800; }
.pill { background: #e6f7ed; color: #008a3e; font-size: 10px; font-weight: bold; padding: 2px 8px; border-radius: 4px; }
.pill.neg { background: #fee2e2; color: #dc2626; }

.bars-container, .segment-container { display: flex; align-items: flex-end; justify-content: space-between; flex: 1; }
.bar-col, .seg-col { display: flex; flex-direction: column; align-items: center; justify-content: flex-end; height: 100%; flex: 1; }
.bar { width: 50%; background: #cfd4d9; border-radius: 2px; }
.bar.orange { background: #f58220; }
.seg-bar { width: 40px; border-radius: 3px; }
.bar-txt { font-size: 9px; font-weight: bold; margin-bottom: 4px; }
.bar-label { font-size: 10px; color: #999; margin-top: 5px; }

.main-viz { min-height: 480px; }
.viz-header { display: flex; justify-content: space-between; }
.viz-legend { display: flex; flex-wrap: wrap; gap: 10px; margin: 15px 0; padding-bottom: 10px; border-bottom: 1px solid #f0f0f0; }
.leg-item { display: flex; align-items: center; gap: 5px; font-size: 10px; cursor: pointer; color: #666; }
.dot { width: 10px; height: 3px; border-radius: 2px; }
.bold-active { color: #f58220; font-weight: bold; }

.viz-body { display: flex; gap: 20px; flex: 1; }
.svg-area { flex: 1; }
.sidebar { width: 220px; border-left: 1px solid #eee; padding-left: 15px; }
.side-item { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; cursor: pointer; }
.badge { font-size: 10px; background: #eee; padding: 2px 6px; border-radius: 4px; width: 45px; text-align: center; }
.badge.orange { background: #f58220; color: white; }

.ia-legend { background: #1a1d21; color: white; padding: 15px; border-radius: 10px; display: flex; gap: 15px; align-items: center; }
.ia-ico { color: #f58220; font-size: 20px; }
.ia-tag { background: #3d3429; color: #f58220; font-size: 9px; padding: 2px 8px; border-radius: 4px; }
.svg-txt { font-size: 11px; fill: #aaa; }
.bar-val { font-size: 10px; fill: #666; font-weight: bold; }
</style>
