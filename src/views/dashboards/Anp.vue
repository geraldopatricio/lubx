<script setup>
import { ref, computed, reactive, onMounted } from 'vue';

// --- CONFIGURAÇÃO DE MARCAS E LOGOS ---
const marcasConfig = {
  TOTALENERGIES: { cor: '#8C6239', logo: 'totalenergies' },
  LUBRAX: { cor: '#008542', logo: 'lubrax' },
  SHELL: { cor: '#FBCE07', logo: 'shell' },
  MOBIL: { cor: '#E10600', logo: 'mobil' },
  CASTROL: { cor: '#008144', logo: 'castrol' },
  YPF: { cor: '#00529B', logo: 'ypf' },
  PETRONAS: { cor: '#00A79D', logo: 'petronas' },
  VALVOLINE: { cor: '#2FA4E7', logo: 'valvoline' },
  IPIRANGA: { cor: '#FFD400', logo: 'ipiranga' },
};

const montadorasConfig = {
  volkswagen: { logo: 'vw' },
  nissan: { logo: 'nissan' },
  honda: { logo: 'honda' },
  fiat: { logo: 'fiat' },
  mitsubishi: { logo: 'mitsubishi' },
  bmw: { logo: 'bmw' },
  ford: { logo: 'ford' },
  toyota: { logo: 'toyota' },
  audi: { logo: 'audi' },
  'mercedes-benz': { logo: 'mercedes' },
};

// --- ESTADO REATIVO ---
const loading = ref(false);
const searchQuery = ref('');
const filtroPrincipal = ref('Indústria'); // 'Indústria' ou 'Montadora'
const showPeriodoModal = ref(false);
const showFiltroDropdown = ref(false);
const expandedRow = ref(null); // ID da linha expandida
const selectedCell = ref(null); // Para o modal de produtos

const state = reactive({
  dataDe: '2025-02-01',
  dataAte: '2025-03-02',
});

// --- NOVOS ESTADOS PARA SIDEBAR ---
const activeSidebarTab = ref('homologacoes'); // 'homologacoes' ou 'indicadores'
const indicadoresFinanceiros = reactive({
  usd: '0,00',
  usdVar: '0,00%',
  eur: '0,00',
  eurVar: '0,00%'
});

// Busca de moedas (API Gratuita AwesomeAPI)
async function fetchIndicadores() {
  try {
    const res = await fetch('https://economia.awesomeapi.com.br/last/USD-BRL,EUR-BRL');
    const data = await res.json();
    indicadoresFinanceiros.usd = parseFloat(data.USDBRL.bid).toLocaleString('pt-BR', { minimumFractionDigits: 2 });
    indicadoresFinanceiros.usdVar = parseFloat(data.USDBRL.pctChange).toFixed(2) + '%';
    indicadoresFinanceiros.eur = parseFloat(data.EURBRL.bid).toLocaleString('pt-BR', { minimumFractionDigits: 2 });
    indicadoresFinanceiros.eurVar = parseFloat(data.EURBRL.pctChange).toFixed(2) + '%';
  } catch (e) {
    console.error("Erro ao carregar indicadores", e);
  }
}

onMounted(() => {
  fetchIndicadores();
});

// --- MOCK DE DADOS (Baseado nos prints) ---
const mockDados = [
  { id: 1, nome: 'TOTALENERGIES', tipo: 'Indústria', valores: { 2022: 1215, 2023: 1687, 2024: 900, 2025: 1578, 2026: 500 }, 
    detalhes: [
      { nome: 'volkswagen', valores: { 2022: 600, 2023: 500, 2024: 200, 2025: 340, 2026: 250 } },
      { nome: 'nissan', valores: { 2022: 307, 2023: 500, 2024: 300, 2025: 540, 2026: 125 } },
      { nome: 'honda', valores: { 2022: 308, 2023: 687, 2024: 400, 2025: 698, 2026: 125 } },
    ]
  },
  { id: 2, nome: 'LUBRAX', tipo: 'Indústria', valores: { 2022: 987, 2023: 4805, 2024: 457, 2025: 1231, 2026: 1564 }, detalhes: [] },
  { id: 3, nome: 'SHELL', tipo: 'Indústria', valores: { 2022: 4805, 2023: 5134, 2024: 1123, 2025: 4798, 2026: 898 }, detalhes: [] },
  { id: 4, nome: 'MOBIL', tipo: 'Indústria', valores: { 2022: 6878, 2023: 2312, 2024: 3423, 2025: 7678, 2026: 464 }, detalhes: [] },
  { id: 5, nome: 'CASTROL', tipo: 'Indústria', valores: { 2022: 2500, 2023: 776, 2024: 5453, 2025: 4805, 2026: 665 }, detalhes: [] },
];

const mockMontadoras = [
  { id: 10, nome: 'volkswagen', tipo: 'Montadora', valores: { 2022: 1215, 2023: 1687, 2024: 900, 2025: 1578, 2026: 500 }, 
    detalhes: [
      { nome: 'TOTALENERGIES', valores: { 2022: 600, 2023: 500, 2024: 200, 2025: 340, 2026: 250 } },
      { nome: 'LUBRAX', valores: { 2022: 307, 2023: 500, 2024: 300, 2025: 540, 2026: 125 } },
    ]
  },
  { id: 11, nome: 'nissan', tipo: 'Montadora', valores: { 2022: 987, 2023: 4805, 2024: 457, 2025: 1231, 2026: 1564 }, detalhes: [] },
  { id: 12, nome: 'honda', tipo: 'Montadora', valores: { 2022: 4805, 2023: 5134, 2024: 1123, 2025: 4798, 2026: 898 }, detalhes: [] },
  { id: 13, nome: 'fiat', tipo: 'Montadora', valores: { 2022: 6878, 2023: 2312, 2024: 3423, 2025: 7678, 2026: 464 }, detalhes: [] },
];

const mockProdutos = [
  { nome: 'Total Quartz 900 5W-40', norma: 'VW 502 00', categoria: 'Motor' },
  { nome: 'Total Quartz Ineo First 0W-30', norma: 'VW 502 00', categoria: 'Motor' },
  { nome: 'Total Quartz Ineo Long Life 5W-30', norma: 'VW 502 00', categoria: 'Motor' },
];

// --- LÓGICA ---
const dadosFiltrados = computed(() => {
  const lista = filtroPrincipal.value === 'Indústria' ? mockDados : mockMontadoras;
  if (!searchQuery.value) return lista;
  return lista.filter(i => i.nome.toLowerCase().includes(searchQuery.value.toLowerCase()));
});

function toggleRow(id) {
  expandedRow.value = expandedRow.value === id ? null : id;
}

function openCellDetails(nomeItem, ano, valor) {
  if (valor > 0) {
    selectedCell.value = { nomeItem, ano, valor };
  }
}

function closeModals() {
  selectedCell.value = null;
  showPeriodoModal.value = false;
  showFiltroDropdown.value = false;
}

const getLogo = (nome) => {
  const n = nome.toUpperCase();
  if (marcasConfig[n]) return `/logos/lubrificantes/${marcasConfig[n].logo}.png`;
  if (montadorasConfig[nome.toLowerCase()]) return `/logos/montadoras/${montadorasConfig[nome.toLowerCase()].logo}.png`;
  return '/logos/lubrificantes/generic.png';
};
</script>

<template>
  <div class="anp-dashboard" @click="closeModals">
    <!-- HEADER SUPERIOR -->
    <header class="top-header">
      <div class="stats">
        <strong>Todas as {{ filtroPrincipal === 'Indústria' ? 'indústrias' : 'montadoras' }}:</strong>
        <span class="highlight"> 891 produtos registrados</span>
      </div>
      <div class="search-bar">
        <i class="search-icon">🔍</i>
        <input v-model="searchQuery" type="text" placeholder="Pesquisar por..." @click.stop />
      </div>
    </header>

    <!-- FILTROS -->
    <div class="filters-row">
      <div class="custom-select" @click.stop="showFiltroDropdown = !showFiltroDropdown">
        <div class="select-trigger">
          {{ filtroPrincipal }} <i class="arrow-down"></i>
        </div>
        <div v-if="showFiltroDropdown" class="select-dropdown">
          <label @click="filtroPrincipal = 'Indústria'">
            <input type="checkbox" :checked="filtroPrincipal === 'Indústria'"> Indústria
          </label>
          <label @click="filtroPrincipal = 'Montadora'">
            <input type="checkbox" :checked="filtroPrincipal === 'Montadora'"> Montadora
          </label>
        </div>
      </div>

      <div class="custom-select" @click.stop="showPeriodoModal = !showPeriodoModal">
        <div class="select-trigger">
          📅 01/02 - 02/03/2025 <i class="arrow-down"></i>
        </div>
        <!-- Modal de Período Mockado conforme Print 4 -->
        <div v-if="showPeriodoModal" class="periodo-modal" @click.stop>
          <div class="periodo-options">
            <button>Hoje</button>
            <button>Últimos 7 dias</button>
            <button>Últimos 30 dias</button>
            <button>Este mês</button>
            <button class="active">Personalizado ✓</button>
          </div>
          <div class="calendar-view">
            <div class="cal"><span>Março, 2023</span><div class="cal-placeholder"></div></div>
            <div class="cal"><span>Março, 2026</span><div class="cal-placeholder"></div></div>
          </div>
          <button class="btn-apply">Aplicar período</button>
        </div>
      </div>
    </div>

    <!-- CONTEÚDO PRINCIPAL -->
    <div class="main-content">
      <!-- TABELA -->
      <section class="table-container">
        <div class="table-header-info">
          <span>Marca/Ano/Produto</span>
          <span class="reg-count">24 Registros</span>
          <button class="btn-export">Exportar 📥</button>
        </div>

        <table class="anp-table">
          <thead>
            <tr>
              <th class="text-left">{{ filtroPrincipal.toUpperCase() }} > {{ filtroPrincipal === 'Indústria' ? 'MONTADORA' : 'INDÚSTRIA' }}</th>
              <th>2022</th>
              <th>2023</th>
              <th>2024</th>
              <th>2025</th>
              <th>2026</th>
            </tr>
          </thead>
          <tbody>
            <template v-for="item in dadosFiltrados" :key="item.id">
              <!-- Linha Principal -->
              <tr class="main-row" :class="{ 'is-expanded': expandedRow === item.id }">
                <td class="name-cell" @click="toggleRow(item.id)">
                  <img :src="getLogo(item.nome)" class="logo-img" />
                  <span class="item-nome">{{ item.nome }}</span>
                  <i class="arrow-icon" :class="{ 'up': expandedRow === item.id }"></i>
                </td>
                <td v-for="(val, ano) in item.valores" :key="ano">{{ val }}</td>
              </tr>

              <!-- Linhas de Detalhe (Drill-down) -->
              <template v-if="expandedRow === item.id">
                <tr v-for="det in item.detalhes" :key="det.nome" class="sub-row">
                  <td class="sub-name-cell">
                    <img :src="getLogo(det.nome)" class="logo-sub-img" />
                    <span>{{ det.nome }}</span>
                  </td>
                  <td v-for="(v, a) in det.valores" :key="a" 
                      :class="{ 'clickable-cell': v > 0 }" 
                      @click.stop="openCellDetails(det.nome, a, v)">
                    <span v-if="v > 0" class="val-badge">{{ v }}</span>
                    <span v-else>{{ v }}</span>
                  </td>
                </tr>
              </template>
            </template>
          </tbody>
        </table>

        <!-- Paginação fake -->
        <div class="pagination">
          <i class="prev">‹</i> <span>01</span> <span>02</span> <span>03</span> ...35 <i class="next">›</i>
        </div>
      </section>

      <!-- SIDEBAR NEWS (SOLICITAÇÃO DE AJUSTE AQUI) -->
      <aside class="lubx-news">
        <div class="news-header">
          <img src="/Lubx_White.png" alt="LUBX" class="news-logo" />
          <span>News Mercado Automotivo</span>
        </div>
        <div class="news-card-dark">
          <!-- TABS DO SIDEBAR -->
          <div class="sidebar-tabs">
            <button :class="{ active: activeSidebarTab === 'homologacoes' }" @click="activeSidebarTab = 'homologacoes'">🔥 Novas homologações</button>
            <button :class="{ active: activeSidebarTab === 'indicadores' }" @click="activeSidebarTab = 'indicadores'">📊 Indicadores</button>
          </div>

          <!-- ABA: NOVAS HOMOLOGAÇÕES (Print 2 layout) -->
          <div v-if="activeSidebarTab === 'homologacoes'">
            <h4>Acompanhe em primeira mão os novos produtos homologados</h4>
            <p>Informações oficiais direto da indústria, para a indústria.</p>
            <div class="search-news">
              <i class="search-icon-small">🔍</i>
              <input type="text" placeholder="Buscar produto, fabricante ou especificações....">
            </div>

            <div class="homolog-card-v2">
              <div class="card-date-v2">Hoje • 27 de março</div>
              <div class="card-title-v2">TOTAL QUARTZ INEO XTRA</div>
              <div class="card-grid">
                <div class="grid-item">
                  <i class="icon-gray">🏢</i>
                  <span class="label-gray">Fabricante</span>
                  <strong class="val-white">TotalEnergies</strong>
                </div>
                <div class="grid-item">
                  <i class="icon-gray">🛡️</i>
                  <span class="label-gray">Detentor</span>
                  <div class="detentor-badge">
                    <span class="dot green"></span>
                    <strong class="val-white">Indústria</strong>
                  </div>
                </div>
                <div class="grid-item">
                  <i class="icon-gray">📝</i>
                  <span class="label-gray">Especificações</span>
                  <strong class="val-white">API SP / ILSAC GF - 6A</strong>
                </div>
                <div class="grid-item">
                  <i class="icon-gray">📅</i>
                  <span class="label-gray">Homologado em</span>
                  <strong class="val-white">27 Mar 2026</strong>
                </div>
              </div>
            </div>

            <div class="homolog-card-v2">
              <div class="card-date-v2">Ontem • 26 de março</div>
              <div class="card-title-v2">TOTAL QUARTZ 7000</div>
              <div class="card-grid">
                <div class="grid-item">
                  <i class="icon-gray">🏢</i>
                  <span class="label-gray">Fabricante</span>
                  <strong class="val-white">TotalEnergies</strong>
                </div>
                <div class="grid-item">
                  <i class="icon-gray">🛡️</i>
                  <span class="label-gray">Detentor</span>
                  <div class="detentor-badge">
                    <span class="dot yellow"></span>
                    <strong class="val-white">Montadora</strong>
                  </div>
                </div>
                <div class="grid-item">
                  <i class="icon-gray">📝</i>
                  <span class="label-gray">Especificações</span>
                  <strong class="val-white">API SP / ILSAC GF - 6A</strong>
                </div>
                <div class="grid-item">
                  <i class="icon-gray">📅</i>
                  <span class="label-gray">Homologado em</span>
                  <strong class="val-white">27 Mar 2026</strong>
                </div>
              </div>
            </div>
          </div>

          <!-- ABA: INDICADORES (LUMEN POSTO CLUB) -->
          <div v-if="activeSidebarTab === 'indicadores'" class="indicators-view">
            <div class="indicador-header-lumen">
               <span class="lumen-tag">📊 POSTO EM DIA</span>
               <span class="lumen-date">Quinta · 23 abr 2026</span>
            </div>
            
            <div class="indicadores-list">
              <!-- Petroleo -->
              <div class="ind-group">
                <div class="ind-group-title">🛢️ PETRÓLEO</div>
                <div class="ind-line"><span>Brent</span> <strong>US$ 101,55 <small class="text-up">+3,5% ▲</small></strong></div>
                <div class="ind-line"><span>WTI</span> <strong>US$ 92,46 <small class="text-up">+3,1% ▲</small></strong></div>
              </div>

              <!-- Moedas Real-time -->
              <div class="ind-group">
                <div class="ind-group-title">💵 MOEDAS (Real-time)</div>
                <div class="ind-line"><span>USD/BRL</span> <strong>R$ {{ indicadoresFinanceiros.usd }} <small :class="indicadoresFinanceiros.usdVar.includes('-') ? 'text-down' : 'text-up'">{{ indicadoresFinanceiros.usdVar }}</small></strong></div>
                <div class="ind-line"><span>EUR/BRL</span> <strong>R$ {{ indicadoresFinanceiros.eur }} <small :class="indicadoresFinanceiros.eurVar.includes('-') ? 'text-down' : 'text-up'">{{ indicadoresFinanceiros.eurVar }}</small></strong></div>
                <div class="ind-line"><span>Ibovespa</span> <strong>197.324 <small class="text-up">máx hist ▲</small></strong></div>
              </div>

              <!-- ANP -->
              <div class="ind-group">
                <div class="ind-group-title">⛽ ANP SÍNTESE SEM</div>
                <div class="ind-line"><span>Gasolina C</span> <strong>R$ 6,29/L</strong></div>
                <div class="ind-line"><span>Diesel S-10</span> <strong>R$ 7,58/L</strong></div>
              </div>

              <div class="radar-box">
                <strong class="radar-title">🛰️ RADAR DO DIA</strong>
                <p>Brent disparou +3,5% p/ US$ 101. Pressão máxima sobre Petrobras p/ reajuste.</p>
              </div>
            </div>
          </div>
        </div>
      </aside>
    </div>

    <!-- MODAL DE PRODUTOS (Print 3) -->
    <div v-if="selectedCell" class="product-modal-overlay" @click="selectedCell = null">
      <div class="product-modal" @click.stop>
        <div class="modal-label">Produtos</div>
        <div class="product-list">
          <div v-for="p in mockProdutos" :key="p.nome" class="product-row">
            <span class="p-name">{{ p.nome }}</span>
            <span class="p-tag gray">{{ p.norma }}</span>
            <span class="p-tag gray">{{ p.categoria }}</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.anp-dashboard {
  background: #f4f4f4;
  min-height: 100vh;
  padding: 20px;
  font-family: 'Segoe UI', Roboto, sans-serif;
  color: #333;
}

/* TOP HEADER */
.top-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: white;
  padding: 15px 25px;
  border-radius: 12px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}
.highlight { color: #f58220; font-weight: 700; }
.search-bar {
  background: #f9f9f9;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  padding: 8px 15px;
  display: flex;
  align-items: center;
  width: 300px;
}
.search-bar input { border: none; background: transparent; outline: none; margin-left: 10px; width: 100%; }

/* FILTROS */
.filters-row { display: flex; gap: 15px; margin: 20px 0; }
.custom-select {
  background: white;
  padding: 10px 15px;
  border-radius: 8px;
  border: 1px solid #ddd;
  cursor: pointer;
  position: relative;
  min-width: 180px;
  font-size: 14px;
}
.select-trigger { display: flex; justify-content: space-between; align-items: center; }
.select-dropdown {
  position: absolute;
  top: 100%; left: 0;
  background: white;
  width: 100%;
  border: 1px solid #ddd;
  border-radius: 8px;
  z-index: 10;
  padding: 10px;
}
.select-dropdown label { display: block; padding: 5px 0; cursor: pointer; }

/* PERIODO MODAL (PRINT 4) */
.periodo-modal {
  position: absolute;
  top: 110%; left: 0;
  background: white;
  width: 500px;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.2);
  z-index: 100;
}
.periodo-options { display: flex; flex-direction: column; gap: 10px; width: 150px; float: left; border-right: 1px solid #eee; }
.periodo-options button { background: none; border: none; text-align: left; padding: 5px; cursor: pointer; color: #666; }
.periodo-options button.active { color: #f58220; font-weight: bold; }
.calendar-view { margin-left: 170px; display: flex; gap: 20px; }
.cal-placeholder { height: 150px; width: 100%; background: #fcfcfc; border: 1px dashed #ccc; margin-top: 10px; }
.btn-apply { background: #1a1a1a; color: white; border: none; width: 100%; padding: 10px; border-radius: 6px; margin-top: 20px; cursor: pointer; }

/* MAIN CONTENT LAYOUT */
.main-content { display: grid; grid-template-columns: 1fr 350px; gap: 20px; }

/* TABLE */
.table-container { background: white; border-radius: 12px; padding: 20px; }
.table-header-info { display: flex; align-items: center; gap: 20px; margin-bottom: 15px; color: #888; font-size: 13px; }
.btn-export { background: #f58220; color: white; border: none; padding: 8px 15px; border-radius: 6px; cursor: pointer; margin-left: auto; font-weight: bold; }

.anp-table { width: 100%; border-collapse: collapse; }
.anp-table th { text-align: center; padding: 12px; color: #8C6239; font-size: 14px; border-bottom: 2px solid #f0f0f0; }
.anp-table th.text-left { text-align: left; }
.anp-table td { padding: 12px; text-align: center; border-bottom: 1px solid #f9f9f9; font-size: 14px; }

.main-row { cursor: pointer; transition: background 0.2s; }
.main-row:hover { background: #fffaf5; }
.main-row.is-expanded { background: #fff6ed; }

.name-cell { display: flex; align-items: center; gap: 12px; text-align: left !important; font-weight: 600; }
.logo-img { width: 24px; height: 24px; object-fit: contain; }
.arrow-icon { margin-left: auto; border: solid #666; border-width: 0 2px 2px 0; display: inline-block; padding: 3px; transform: rotate(45deg); transition: 0.3s; }
.arrow-icon.up { transform: rotate(-135deg); }

/* SUB ROWS */
.sub-row { background: #fcfcfc; }
.sub-name-cell { display: flex; align-items: center; gap: 10px; padding-left: 40px !important; text-align: left !important; color: #666; font-size: 13px; }
.logo-sub-img { width: 18px; height: 18px; filter: grayscale(1); }

.val-badge { background: #fff3e6; color: #f58220; padding: 4px 10px; border-radius: 4px; font-weight: bold; }
.clickable-cell { cursor: pointer; }

/* --- NOVOS ESTILOS SIDEBAR (SOLICITADOS) --- */
.lubx-news { background: #1a1a1a; border-radius: 15px; padding: 20px; color: white; overflow: hidden; }
.news-header { display: flex; align-items: center; gap: 10px; margin-bottom: 20px; font-size: 11px; color: #777; text-transform: uppercase; letter-spacing: 1px;}
.news-logo { height: 18px; }

.sidebar-tabs { display: flex; background: #262626; padding: 4px; border-radius: 8px; margin-bottom: 20px; }
.sidebar-tabs button { flex: 1; border: none; background: transparent; color: #777; padding: 8px; font-size: 11px; font-weight: bold; cursor: pointer; border-radius: 6px; transition: 0.2s; }
.sidebar-tabs button.active { background: #3d2b1a; color: #f58220; }

.news-card-dark h4 { margin: 0 0 8px; font-size: 16px; line-height: 1.3; }
.news-card-dark p { font-size: 12px; color: #777; margin-bottom: 20px; }
.search-news { position: relative; width: 100%; margin-bottom: 20px; }
.search-news input { width: 100%; background: #262626; border: 1px solid #333; padding: 10px 10px 10px 30px; border-radius: 8px; color: white; font-size: 12px; outline: none; }
.search-icon-small { position: absolute; left: 10px; top: 50%; transform: translateY(-50%); color: #555; font-size: 12px; }

/* CARD V2 (IGUAL AO PRINT 2) */
.homolog-card-v2 { background: #222; border: 1px solid #333; border-radius: 12px; padding: 15px; margin-bottom: 15px; border-left: 4px solid #8C6239; position: relative;}
.card-date-v2 { font-size: 11px; color: #f58220; margin-bottom: 12px; background: #3d2b1a; display: inline-block; padding: 4px 10px; border-radius: 20px; }
.card-title-v2 { font-weight: bold; font-size: 14px; margin-bottom: 15px; color: #fff; letter-spacing: 0.5px;}

.card-grid { display: flex; flex-direction: column; gap: 12px; }
.grid-item { display: grid; grid-template-columns: 25px 100px 1fr; align-items: center; font-size: 11px; }
.icon-gray { opacity: 0.5; font-style: normal; }
.label-gray { color: #888; }
.val-white { color: #fff; font-weight: bold; text-align: right; }

.detentor-badge { display: flex; align-items: center; gap: 6px; background: #2a2a2a; padding: 4px 10px; border-radius: 6px; justify-self: end; }
.dot { width: 7px; height: 7px; border-radius: 50%; }
.dot.green { background: #00ff88; }
.dot.yellow { background: #ffcc00; }

/* ABA INDICADORES */
.indicators-view { font-size: 12px; }
.indicador-header-lumen { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #333; padding-bottom: 10px; margin-bottom: 15px; }
.lumen-tag { font-weight: bold; color: #f58220; }
.lumen-date { color: #777; font-size: 10px; }

.ind-group { margin-bottom: 18px; }
.ind-group-title { font-size: 10px; font-weight: bold; color: #555; border-left: 2px solid #f58220; padding-left: 8px; margin-bottom: 8px; }
.ind-line { display: flex; justify-content: space-between; padding: 5px 0; border-bottom: 1px solid #222; }
.ind-line span { color: #888; }
.ind-line strong { color: #fff; }
.text-up { color: #00ff88; font-size: 10px; margin-left: 5px; }
.text-down { color: #ff4444; font-size: 10px; margin-left: 5px; }

.radar-box { background: #222; border: 1px dashed #444; border-radius: 8px; padding: 12px; margin-top: 15px; }
.radar-title { display: block; color: #f58220; margin-bottom: 5px; font-size: 11px; }
.radar-box p { font-size: 11px; color: #aaa; margin: 0; line-height: 1.4; }

/* --- FIM DOS NOVOS ESTILOS --- */

/* MODAL PRODUTOS (PRINT 3) */
.product-modal-overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.4); display: flex; align-items: center; justify-content: center; z-index: 1000; }
.product-modal { background: white; width: 450px; border-radius: 15px; padding: 25px; box-shadow: 0 20px 50px rgba(0,0,0,0.3); }
.modal-label { font-size: 12px; font-weight: bold; color: #888; margin-bottom: 15px; }
.product-row { display: flex; gap: 10px; align-items: center; margin-bottom: 10px; }
.p-name { background: #eee; padding: 6px 12px; border-radius: 20px; font-size: 12px; flex: 1; }
.p-tag { background: #f5f5f5; border: 1px solid #ddd; padding: 6px 12px; border-radius: 8px; font-size: 11px; color: #666; }

.pagination { display: flex; justify-content: center; gap: 15px; margin-top: 30px; color: #888; font-size: 14px; align-items: center; }
</style>
