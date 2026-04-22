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

      <!-- SIDEBAR NEWS -->
      <aside class="lubx-news">
        <div class="news-header">
          <img src="/Lubx_White.png" alt="LUBX" class="news-logo" />
          <span>News Mercado Automotivo</span>
        </div>
        <div class="news-card-dark">
          <span class="tag-orange">🔥 Novas homologações</span>
          <h4>Acompanhe em primeira mão os novos produtos homologados</h4>
          <p>Informações oficiais direto da indústria, para a indústria.</p>
          <div class="search-news">
             <input type="text" placeholder="Buscar produto, fabricante...">
          </div>

          <!-- Cards de Homologação (Print 1) -->
          <div class="homolog-card">
            <div class="card-date">Hoje • 27 de março</div>
            <div class="card-title">TOTAL QUARTZ INEO XTRA</div>
            <div class="card-info">
              <div class="info-line"><span>Fabricante</span> <strong>TotalEnergies</strong></div>
              <div class="info-line"><span>Detentor</span> <strong class="badge-ind">● Indústria</strong></div>
              <div class="info-line"><span>Especificações</span> <strong>API SP / ILSAC GF - 6A</strong></div>
              <div class="info-line"><span>Homologado em</span> <strong>27 Mar 2026</strong></div>
            </div>
          </div>

          <div class="homolog-card">
            <div class="card-date">Ontem • 26 de março</div>
            <div class="card-title">TOTAL QUARTZ 7000</div>
            <div class="card-info">
              <div class="info-line"><span>Fabricante</span> <strong>TotalEnergies</strong></div>
              <div class="info-line"><span>Detentor</span> <strong class="badge-mon">● Montadora</strong></div>
            </div>
          </div>

          <button class="btn-newsletter">Assinar Newsletter ✉</button>
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

/* SIDEBAR */
.lubx-news { background: #222; border-radius: 15px; padding: 20px; color: white; }
.news-header { display: flex; align-items: center; gap: 10px; margin-bottom: 20px; font-size: 12px; opacity: 0.8; }
.news-logo { height: 20px; }
.tag-orange { background: #4d2b00; color: #ff9900; padding: 4px 8px; border-radius: 4px; font-size: 10px; font-weight: bold; }
.news-card-dark h4 { margin: 15px 0 10px; font-size: 18px; line-height: 1.3; }
.news-card-dark p { font-size: 13px; color: #aaa; margin-bottom: 20px; }
.search-news input { width: 100%; background: #333; border: 1px solid #444; padding: 10px; border-radius: 8px; color: white; margin-bottom: 20px; }

.homolog-card { background: #1a1a1a; border: 1px solid #333; border-radius: 12px; padding: 15px; margin-bottom: 15px; border-left: 3px solid #f58220; }
.card-date { font-size: 10px; color: #f58220; margin-bottom: 10px; }
.card-title { font-weight: bold; font-size: 14px; margin-bottom: 10px; color: #eee; }
.info-line { display: flex; justify-content: space-between; font-size: 11px; margin-bottom: 5px; }
.info-line span { color: #888; }
.badge-ind { color: #00ff88; }
.badge-mon { color: #ffcc00; }
.btn-newsletter { width: 100%; background: #f58220; border: none; padding: 12px; border-radius: 8px; color: white; font-weight: bold; cursor: pointer; margin-top: 10px; }

/* MODAL PRODUTOS (PRINT 3) */
.product-modal-overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.4); display: flex; align-items: center; justify-content: center; z-index: 1000; }
.product-modal { background: white; width: 450px; border-radius: 15px; padding: 25px; box-shadow: 0 20px 50px rgba(0,0,0,0.3); }
.modal-label { font-size: 12px; font-weight: bold; color: #888; margin-bottom: 15px; }
.product-row { display: flex; gap: 10px; align-items: center; margin-bottom: 10px; }
.p-name { background: #eee; padding: 6px 12px; border-radius: 20px; font-size: 12px; flex: 1; }
.p-tag { background: #f5f5f5; border: 1px solid #ddd; padding: 6px 12px; border-radius: 8px; font-size: 11px; color: #666; }

.pagination { display: flex; justify-content: center; gap: 15px; margin-top: 30px; color: #888; font-size: 14px; align-items: center; }
</style>
