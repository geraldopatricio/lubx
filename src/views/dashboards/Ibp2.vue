<script setup>
import { ref, reactive, onMounted, watch, computed, nextTick } from 'vue';

const API_BASE = 'https://lubx-api.lubconsulta.com.br/bi/ipb';

const loading = ref(true);
const dashboardData = ref(null);

const state = reactive({
  empresa: ['TOTALENERGIES'],
  ano: '2025',
  dataDe: '2025-01-01',
  dataAte: '2025-12-31',
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

// Função para selecionar o ano e forçar o reset das datas
const selecionarAno = (ano) => {
  state.ano = ano.toString();
  state.dataDe = `${ano}-01-01`;
  state.dataAte = `${ano}-12-31`;
  // O watch(state) cuidará do fetchDashboard
};

const toggleEmpresa = (nome) => {
  const index = state.empresa.indexOf(nome);
  if (index > -1) {
    if (state.empresa.length > 1) state.empresa.splice(index, 1);
  } else {
    state.empresa.push(nome);
  }
};

async function fetchDashboard() {
  loading.value = true;
  try {
    const params = new URLSearchParams();
    
    // Mapeamento explícito para garantir que a API receba os dados
    params.append('ano', state.ano);
    params.append('segmento', state.segmento);
    params.append('dataDe', state.dataDe);
    params.append('dataAte', state.dataAte);
    
    // Envia múltiplas empresas
    state.empresa.forEach(e => params.append('empresa', e));

    const response = await fetch(`${API_BASE}/dashboard?${params.toString()}`);
    const json = await response.json();
    if (json.status === 'ok') {
      dashboardData.value = json.data;
    }
  } catch (error) {
    console.error("Erro ao buscar dados:", error);
  } finally {
    loading.value = false;
  }
}

// Escuta qualquer mudança no estado de forma profunda
watch(state, () => {
  fetchDashboard();
}, { deep: true });

const chartMetrics = computed(() => {
  if (!dashboardData.value?.graficos?.vendasPorMes) return null;

  const todasAsSeries = dashboardData.value.graficos.vendasPorMes.series;
  const todosOsMeses = dashboardData.value.graficos.vendasPorMes.meses;

  // 1. Filtro de Período
  const mesInicioIdx = parseInt(state.dataDe.split('-')[1]) - 1;
  const mesFimIdx = parseInt(state.dataAte.split('-')[1]) - 1;
  const mesesFiltrados = todosOsMeses.slice(mesInicioIdx, mesFimIdx + 1);

  // 2. Cálculo do Volume Total do Mercado por Mês (Barras Cinzas)
  const marketVolumes = mesesFiltrados.map((_, i) => {
    const realIdx = i + mesInicioIdx;
    return todasAsSeries.reduce((sum, s) => sum + (s.valores[realIdx] || 0), 0);
  });

  const maxMarketVolume = Math.max(...marketVolumes, 1);

  // 3. Configuração de Layout SVG
  const startX = 80; // Recuo para caber os números do eixo Y
  const gap = mesesFiltrados.length > 1 ? 850 / (mesesFiltrados.length - 1) : 0;

  // 4. Mapeamento das Linhas como % de Share (0% a 100%)
  const allLines = todasAsSeries.map(s => ({
    empresa: s.empresa,
    cor: marcasConfig[s.empresa]?.cor || '#ccc',
    destaque: state.empresa.includes(s.empresa),
    points: s.valores.slice(mesInicioIdx, mesFimIdx + 1).map((v, i) => {
      const totalMes = marketVolumes[i] || 1;
      const share = (v / totalMes) * 100; // Cálculo do % de Share
      return {
        x: i * gap + startX,
        y: 280 - (share / 100 * 220), // Mapeia 100% share para o topo do gráfico (220px de altura)
        shareLabel: share.toFixed(1) + '%'
      };
    })
  }));

  // 5. Mapeamento das Barras (Volume Total Mercado)
  const bars = marketVolumes.map((v, i) => ({
    x: i * gap + startX - 20,
    h: (v / maxMarketVolume) * 220, // Normalizado pelo maior volume do período
    label: `R$ ${Math.round(v / 1000)}k`
  }));

  return { meses: mesesFiltrados, bars, allLines, gap, startX };
});

const formatDateDisplay = (dateStr) => {
  if (!dateStr) return '';
  const [y, m, d] = dateStr.split('-');
  return `${d}/${m}/${y}`;
};

onMounted(fetchDashboard);

const formatM3 = (v) => v ? v.toLocaleString('pt-BR') + ' m³' : '0 m³';
</script>

<template>
  <div class="pbi-layout" v-if="dashboardData">
    <header class="pbi-filters">
      <div class="filter-left">
        <!-- FILTRO EMPRESAS -->
        <div class="multi-select-container">
          <div class="multi-select-trigger">
             🏢 {{ state.empresa.length }} Marca(s)
          </div>
          <div class="multi-select-dropdown">
            <label v-for="e in dashboardData.filtros.empresas" :key="e" class="checkbox-item">
              <input type="checkbox" :checked="state.empresa.includes(e)" @change="toggleEmpresa(e)">
              <span>{{ e }}</span>
            </label>
          </div>
        </div>

        <!-- FILTRO SEGMENTO -->
        <div class="pbi-tabs">
          <button v-for="s in dashboardData.filtros.segmentos" :key="s.id" 
                  :class="{ active: state.segmento === s.id }" @click="state.segmento = s.id">
            {{ s.label.toUpperCase() }}
          </button>
        </div>

        <!-- FILTRO CALENDÁRIO (Interativo) -->
        <div class="period-picker">
          <div class="date-field">
            <label>DE</label>
            <input type="date" v-model="state.dataDe" 
                   :min="`${state.ano}-01-01`" 
                   :max="`${state.ano}-12-31`" />
          </div>
          <div class="date-field">
            <label>ATÉ</label>
            <input type="date" v-model="state.dataAte" 
                   :min="state.dataDe" 
                   :max="`${state.ano}-12-31`" />
          </div>
        </div>
      </div>

      <button class="btn-clear" @click="selecionarAno('2025')">Limpar Filtros</button>
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

        <!-- Gráfico Anual -->
        <div class="card chart-annual">
          <div class="card-title">Venda Anual</div>
          <div class="bars-container">
            <div v-for="b in dashboardData.graficos.vendaAnual.items" :key="b.ano" class="bar-col">
              <span class="bar-txt">{{ Math.round(b.valor/1000) }}k</span>
              <div class="bar" :class="{ orange: b.ano == state.ano }" 
                   :style="{ height: (b.valor / Math.max(...dashboardData.graficos.vendaAnual.items.map(x=>x.valor), 1) * 100) + '%' }"></div>
              <span class="bar-label">{{ b.ano }}</span>
            </div>
          </div>
        </div>

        <!-- Gráfico Segmento -->
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

      <!-- GRÁFICO PRINCIPAL -->
      <section class="card main-viz">
        <div class="viz-header">
          <div class="viz-info">
            <h3>Vendas de Lubrificantes por Mês</h3>
            <p>Período: {{ formatDateDisplay(state.dataDe) }} a {{ formatDateDisplay(state.dataAte) }}</p>
          </div>
          <div class="viz-actions">
            <div class="tabs-anos">
              <!-- BOTÕES DE ANO COM FUNÇÃO DE SELEÇÃO CORRIGIDA -->
              <button v-for="ano in dashboardData.filtros.anos" :key="ano" 
                      :class="{ active: state.ano == ano }" 
                      @click="selecionarAno(ano)">{{ ano }}</button>
            </div>
          </div>
        </div>

        <div class="viz-legend">
          <div v-for="(cfg, nome) in marcasConfig" :key="nome" class="leg-item" @click="toggleEmpresa(nome)">
            <span class="dot" :style="{ background: cfg.cor }"></span> 
            <span :class="{ 'bold-active': state.empresa.includes(nome) }">{{ nome }}</span>
          </div>
        </div>

        <div class="viz-body">
          <div class="svg-area">
            <svg viewBox="0 0 1000 320" preserveAspectRatio="none">
              <!-- EIXO Y (Numeração de % Share) -->
              <g class="axis-y">
                <text v-for="n in [0, 25, 50, 75, 100]" :key="n" 
                      x="45" :y="280 - (n/100 * 220)" 
                      text-anchor="end" class="svg-txt-axis">{{ n }}%</text>
                <!-- Linhas de grade horizontais -->
                <line v-for="n in [0, 25, 50, 75, 100]" :key="'l'+n" 
                      x1="55" :y1="280 - (n/100 * 220)" 
                      x2="950" :y2="280 - (n/100 * 220)" 
                      stroke="#f0f0f0" stroke-width="1" />
              </g>
              
              <!-- BARRAS (Volume Total do Mercado conforme API) -->
              <g v-for="(b, i) in chartMetrics?.bars" :key="'b'+i">
                <rect :x="b.x" :y="280 - b.h" width="40" :height="b.h" fill="#e9ecef" rx="2" />
                <text :x="b.x + 20" :y="280 - b.h - 10" text-anchor="middle" class="bar-val">{{ b.label }}</text>
              </g>

              <!-- LINHAS (Evolução do % de Share) -->
              <g v-for="line in chartMetrics?.allLines" :key="line.empresa">
                <polyline 
                  :points="line.points.map(p => `${p.x},${p.y}`).join(' ')" 
                  fill="none" 
                  :stroke="line.cor" 
                  :stroke-width="line.destaque ? 4 : 1.5"
                  :stroke-opacity="line.destaque ? 1 : 0.2"
                  stroke-linejoin="round"
                />
                <!-- Pontos e Tooltip de % -->
                <g v-if="line.destaque">
                  <g v-for="(p, i) in line.points" :key="'c'+i">
                    <circle :cx="p.x" :cy="p.y" r="4" :fill="line.cor" />
                    <!-- Label de % sobre o ponto para destaque -->
                    <text :x="p.x" :y="p.y - 8" text-anchor="middle" class="share-label-txt">{{ p.shareLabel }}</text>
                  </g>
                </g>
              </g>

              <!-- EIXO X (Meses) -->
              <text v-for="(m, i) in chartMetrics?.meses" :key="m" 
                    :x="i * (chartMetrics?.gap || 0) + (chartMetrics?.startX || 0)" 
                    y="310" text-anchor="middle" class="svg-txt">{{ m }}</text>
            </svg>
          </div>

          <div class="sidebar">
            <div class="side-title">% Share Mercado</div>
            <div v-for="m in dashboardData.marcas" :key="m.nome" class="side-item" @click="toggleEmpresa(m.nome)">
              <img :src="`/logos/lubrificantes/${marcasConfig[m.nome]?.logo || 'generic'}.png`" class="brand-logo-img" />
              <span class="name" :class="{ 'bold-txt': state.empresa.includes(m.nome) }">{{ m.nome }}</span>
              <span class="badge" :class="{ orange: state.empresa.includes(m.nome) }">24,8%</span>
            </div>
          </div>
        </div>
      </section>

      <footer class="ia-legend">
        <div class="ia-ico">✦</div>
        <div class="ia-content">
          <div class="ia-title">LubAssist IA <span class="ia-tag">Análise Ativa</span></div>
          <p>Exibindo dados de <b>{{ state.empresa.join(' & ') }}</b> para o período selecionado em <b>{{ state.ano }}</b>.</p>
        </div>
      </footer>
    </div>
  </div>
</template>

<style scoped>
/* ESTILOS DOS CALENDÁRIOS */
.period-picker { display: flex; gap: 10px; align-items: center; background: #f8f9fa; padding: 4px 10px; border-radius: 6px; border: 1px solid #ddd; }
.date-field { display: flex; align-items: center; gap: 5px; }
.date-field label { font-size: 9px; font-weight: bold; color: #888; }
.date-field input { border: 1px solid #ccc; border-radius: 4px; padding: 2px 5px; font-size: 11px; outline: none; background: white; }
.date-field input:focus { border-color: #f58220; }

/* AJUSTES DE INTERATIVIDADE */
.tabs-anos button { transition: all 0.2s ease; }
.tabs-anos button:hover { background: #fff3e0; }
.bold-txt { font-weight: bold; color: #f58220; }

/* MANUTENÇÃO DO LAYOUT ORIGINAL */
.multi-select-container { position: relative; width: 160px; background: white; border: 1px solid #ccc; border-radius: 4px; font-size: 12px; cursor: pointer; }
.multi-select-trigger { padding: 8px 12px; font-weight: bold; }
.multi-select-dropdown { position: absolute; top: 100%; left: 0; width: 220px; background: white; border: 1px solid #ccc; max-height: 250px; overflow-y: auto; z-index: 100; display: none; box-shadow: 0 4px 10px rgba(0,0,0,0.1); border-radius: 0 0 4px 4px; }
.multi-select-container:hover .multi-select-dropdown { display: block; }
.checkbox-item { display: flex; align-items: center; gap: 8px; padding: 8px 12px; cursor: pointer; }
.checkbox-item:hover { background: #f9f9f9; }
.pbi-layout { background: #f3f3f3; min-height: 100vh; padding: 15px; font-family: 'Segoe UI', sans-serif; }
.pbi-content { display: flex; flex-direction: column; gap: 15px; min-width: 1200px; transition: opacity 0.3s; }
.loading-opacity { opacity: 0.5; pointer-events: none; }
.pbi-filters { display: flex; justify-content: space-between; background: white; padding: 10px 20px; border-radius: 8px; align-items: center; margin-bottom: 10px; box-shadow: 0 2px 4px rgba(0,0,0,0.05); }
.filter-left { display: flex; align-items: center; gap: 15px; }
.pbi-tabs { display: flex; background: #eee; padding: 3px; border-radius: 6px; }
.pbi-tabs button { border: none; background: transparent; padding: 5px 12px; font-size: 11px; cursor: pointer; font-weight: bold; color: #666; }
.pbi-tabs button.active { background: #f58220; color: white; border-radius: 4px; }
.tabs-anos { display: flex; gap: 5px; }
.tabs-anos button { border: 1px solid #e0e0e0; background: #fff; padding: 5px 12px; font-size: 12px; cursor: pointer; border-radius: 4px; }
.tabs-anos button.active { background: #f58220; color: white; border-color: #f58220; font-weight: bold; }
.btn-clear { background: white; border: 1px solid #333; padding: 7px 15px; border-radius: 4px; font-weight: bold; cursor: pointer; font-size: 11px; }
.top-section { display: grid; grid-template-columns: 3fr 1.5fr 1fr; gap: 15px; height: 260px; }
.kpi-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; }
.card { background: white; border-radius: 10px; border: 1px solid #e0e0e0; padding: 12px; display: flex; flex-direction: column; }
.kpi-val { font-size: 20px; font-weight: 800; margin: 5px 0; }
.pill { background: #e6f7ed; color: #008a3e; font-size: 10px; font-weight: bold; padding: 2px 8px; border-radius: 4px; }
.pill.neg { background: #fee2e2; color: #dc2626; }
.bars-container, .segment-container { display: flex; align-items: flex-end; justify-content: space-between; flex: 1; margin-top: 10px; }
.bar-col, .seg-col { display: flex; flex-direction: column; align-items: center; justify-content: flex-end; height: 100%; flex: 1; }
.bar { width: 50%; background: #cfd4d9; border-radius: 2px; min-height: 2px; }
.bar.orange { background: #f58220; }
.seg-bar { width: 35px; border-radius: 3px; }
.bar-txt { font-size: 9px; font-weight: bold; margin-bottom: 4px; }
.bar-label { font-size: 10px; color: #999; margin-top: 5px; }
.main-viz { min-height: 480px; }
.viz-header { display: flex; justify-content: space-between; align-items: flex-start; }
.viz-legend { display: flex; flex-wrap: wrap; gap: 10px; margin: 15px 0; padding-bottom: 10px; border-bottom: 1px solid #f0f0f0; }
.leg-item { display: flex; align-items: center; gap: 5px; font-size: 10px; cursor: pointer; color: #666; }
.dot { width: 10px; height: 3px; border-radius: 2px; }
.bold-active { color: #f58220; font-weight: bold; }
.viz-body { display: flex; gap: 20px; flex: 1; }
.svg-area { flex: 1; position: relative; }
.sidebar { width: 220px; border-left: 1px solid #eee; padding-left: 15px; }
.side-item { display: flex; align-items: center; gap: 10px; margin-bottom: 8px; cursor: pointer; }
.brand-logo-img { width: 24px; height: 24px; object-fit: contain; }
.badge { font-size: 10px; background: #eee; padding: 2px 6px; border-radius: 4px; width: 45px; text-align: center; }
.badge.orange { background: #f58220; color: white; }
.ia-legend { background: #1a1d21; color: white; padding: 15px; border-radius: 10px; display: flex; gap: 15px; align-items: center; margin-top: 10px; }
.ia-ico { color: #f58220; font-size: 20px; }
.ia-tag { background: #3d3429; color: #f58220; font-size: 9px; padding: 2px 8px; border-radius: 4px; margin-left: 8px; }
.svg-txt { font-size: 11px; fill: #aaa; }
.bar-val { font-size: 9px; fill: #666; font-weight: bold; }

/* Estilo para os números do eixo Y lateral */
.svg-txt-axis {
  font-size: 11px;
  fill: #999;
  font-weight: bold;
}

/* Estilo para a porcentagem que aparece sobre a linha selecionada */
.share-label-txt {
  font-size: 10px;
  fill: #333;
  font-weight: 800;
  pointer-events: none;
}

/* Ajuste na área do SVG para dar espaço ao eixo Y */
.svg-area {
  flex: 1;
  padding-left: 10px;
}

/* Mantém as barras cinzas suaves no fundo */
rect {
  transition: height 0.3s ease;
}

</style>
