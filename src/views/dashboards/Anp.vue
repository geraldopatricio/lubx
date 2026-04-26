<script setup>
import { ref, computed, reactive, onMounted, watch } from 'vue';

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
  volkswagen: { logo: 'vw' }, nissan: { logo: 'nissan' }, honda: { logo: 'honda' },
  fiat: { logo: 'fiat' }, mitsubishi: { logo: 'mitsubishi' }, bmw: { logo: 'bmw' },
  ford: { logo: 'ford' }, toyota: { logo: 'toyota' }, audi: { logo: 'audi' },
  'mercedes-benz': { logo: 'mercedes' },
};

// --- ESTADO REATIVO ---
const API_URL = 'https://lubx-api.lubconsulta.com.br/anp/bi';
const loading = ref(false);
const searchQuery = ref('');
const filtroPrincipal = ref('Indústria'); 
const rawData = ref({}); 

const currentPage = ref(1);
const itemsPerPage = 10;
const anoInicial = ref(2022);
const anoFinal = ref(2026);
const showPeriodoModal = ref(false);
const showFiltroDropdown = ref(false);
const expandedRow = ref(null); 
const selectedCell = ref(null); 
const activeSidebarTab = ref('homologacoes'); 
const indicadoresFinanceiros = reactive({ usd: '0,00', usdVar: '0,00%', eur: '0,00', eurVar: '0,00%' });

// --- INTEGRAÇÃO COM A API ---
async function fetchData() {
  loading.value = true;
  try {
    const visao = filtroPrincipal.value === 'Indústria' ? 'industria' : 'montadora';
    const response = await fetch(`${API_URL}?visao=${visao}&anoInicial=${anoInicial.value}&anoFinal=${anoFinal.value}`);
    const json = await response.json();
    if (json.sucesso) {
      rawData.value = json.dados;
    } else {
      rawData.value = {};
    }
  } catch (e) {
    console.error("Erro API:", e);
    rawData.value = {};
  } finally {
    loading.value = false;
  }
}

// Processa o Objeto da API para Array
const listaFiltrada = computed(() => {
  if (!rawData.value) return [];
  return Object.entries(rawData.value).map(([nome, info]) => ({
    id: nome,
    nome: nome,
    totais: info.totais || {},
    itens: info.itens || {}
  })).filter(item => item.nome.toLowerCase().includes(searchQuery.value.toLowerCase()));
});

const listaPaginada = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage;
  return listaFiltrada.value.slice(start, start + itemsPerPage);
});

const totalPages = computed(() => Math.ceil(listaFiltrada.value.length / itemsPerPage) || 1);

// Lógica para a Sidebar (Novas Homologações)
const newsHomologacoes = computed(() => {
  const all = [];
  Object.entries(rawData.value).forEach(([brandName, brand]) => {
    Object.values(brand.itens).forEach(sub => {
      if(sub.produtos) {
        sub.produtos.forEach(p => all.push({...p, fabricante: brandName}));
      }
    });
  });
  return all.sort((a,b) => b.homologado_em - a.homologado_em).slice(0, 4);
});

// Gera a lista de anos com base na seleção do usuário
const anosExibicao = computed(() => {
  const inicio = parseInt(anoInicial.value);
  const fim = parseInt(anoFinal.value);
  const anos = [];
  
  // Proteção simples para não travar o browser se o usuário inverter os valores
  if (inicio <= fim) {
    for (let i = inicio; i <= fim; i++) {
      anos.push(i);
    }
  }
  return anos;
});

// FUNÇÃO EXPORTAR PARA EXCEL (CSV)
function exportarExcel() {
  // O cabeçalho agora usa a lista dinâmica de anos
  const cabecalho = [filtroPrincipal.value, ...anosExibicao.value];
  
  const linhas = listaFiltrada.value.map(item => {
    // Para cada item, percorremos a lista de anos dinâmica para pegar o valor ou 0
    const valoresAnos = anosExibicao.value.map(ano => item.totais[ano] || 0);
    return [item.nome, ...valoresAnos];
  });

  let csvContent = "data:text/csv;charset=utf-8," 
    + cabecalho.join(";") + "\n"
    + linhas.map(e => e.join(";")).join("\n");

  const encodedUri = encodeURI(csvContent);
  const link = document.createElement("a");
  link.setAttribute("href", encodedUri);
  link.setAttribute("download", `Relatorio_ANP_${filtroPrincipal.value}.csv`);
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

function openProductModal(listaDeItens, ano, nomeEntidade) {
  let produtos = [];
  if (typeof listaDeItens === 'object' && !Array.isArray(listaDeItens)) {
    Object.values(listaDeItens).forEach(sub => {
       produtos.push(...sub.produtos.filter(p => p.homologado_em == ano));
    });
  } else {
    produtos = listaDeItens.filter(p => p.homologado_em == ano);
  }
  if (produtos.length > 0) {
    selectedCell.value = { nome: nomeEntidade, ano, produtos };
  }
}

async function fetchIndicadores() {
  try {
    const res = await fetch('https://economia.awesomeapi.com.br/last/USD-BRL,EUR-BRL');
    const data = await res.json();
    indicadoresFinanceiros.usd = parseFloat(data.USDBRL.bid).toLocaleString('pt-BR', { minimumFractionDigits: 2 });
    indicadoresFinanceiros.usdVar = parseFloat(data.USDBRL.pctChange).toFixed(2) + '%';
    indicadoresFinanceiros.eur = parseFloat(data.EURBRL.bid).toLocaleString('pt-BR', { minimumFractionDigits: 2 });
    indicadoresFinanceiros.eurVar = parseFloat(data.EURBRL.pctChange).toFixed(2) + '%';
  } catch (e) { console.error(e); }
}

onMounted(() => { fetchData(); fetchIndicadores(); });

watch([filtroPrincipal, anoInicial, anoFinal], () => { 
  currentPage.value = 1;
  expandedRow.value = null; 
  fetchData(); 
});

const getLogo = (nome) => {
  const n = (nome || '').toUpperCase();
  if (marcasConfig[n]) return `/logos/lubrificantes/${marcasConfig[n].logo}.png`;
  const mKey = (nome || '').toLowerCase();
  if (montadorasConfig[mKey]) return `/logos/montadoras/${montadorasConfig[mKey].logo}.png`;
  
  // Retorna null em vez de uma imagem genérica
  return null; 
};
</script>

<template>
  <div class="anp-dashboard" @click="selectedCell = null; showPeriodoModal = false; showFiltroDropdown = false;">
    
    <!-- HEADER SUPERIOR -->
    <header class="top-header">
      <div class="stats">
        <strong>Todas as {{ filtroPrincipal === 'Indústria' ? 'indústrias' : 'montadoras' }}:</strong>
        <span class="highlight">&nbsp; {{ listaFiltrada.length }} registros filtrados</span>
      </div>
      <div class="search-bar">
        <i class="search-icon">🔍</i>
        <input v-model="searchQuery" type="text" placeholder="Pesquisar por..." @click.stop />
      </div>
    </header>

    <!-- FILTROS -->
    <div class="filters-row">
      <div class="custom-select" @click.stop="showFiltroDropdown = !showFiltroDropdown">
        <div class="select-trigger">{{ filtroPrincipal }} <i class="arrow-down"></i></div>
        <div v-if="showFiltroDropdown" class="select-dropdown">
          <label class="option-label" @click="filtroPrincipal = 'Indústria'">
            <input type="checkbox" :checked="filtroPrincipal === 'Indústria'"> Indústria
          </label>
          <label class="option-label" @click="filtroPrincipal = 'Montadora'">
            <input type="checkbox" :checked="filtroPrincipal === 'Montadora'"> Montadora
          </label>
        </div>
      </div>
      
      <div class="custom-select" @click.stop="showPeriodoModal = !showPeriodoModal">
        <div class="select-trigger">📅 Do Ano {{ anoInicial }} ao Ano {{ anoFinal }} <i class="arrow-down"></i></div>
        <div v-if="showPeriodoModal" class="periodo-modal" @click.stop>
          <div class="range-inputs">
            <div><label>Início</label><input type="number" v-model.number="anoInicial" min="2020" max="2026"></div>
            <div><label>Fim</label><input type="number" v-model.number="anoFinal" min="2020" max="2026"></div>
          </div>
          <button class="btn-apply" @click="showPeriodoModal = false">Aplicar Filtro</button>
        </div>
      </div>
    </div>

    <div class="main-content">
      <section class="table-container">
        <div class="table-header-info">
          <span>Marca/Ano/Produto | <small>Página {{ currentPage }} de {{ totalPages }}</small></span>
          <button class="btn-export" @click.stop="exportarExcel">Exportar CSV 📥</button>
        </div>

        <table class="anp-table">
          <thead>
            <tr>
                <th class="text-left">
                    {{ filtroPrincipal.toUpperCase() }} > {{ filtroPrincipal === 'Indústria' ? 'MONTADORA' : 'INDÚSTRIA' }}
                </th>
                <!-- DINÂMICO AQUI -->
                <th v-for="ano in anosExibicao" :key="ano">{{ ano }}</th>
            </tr>
            </thead>
          <tbody v-if="!loading">
            <template v-for="item in listaPaginada" :key="item.id">
              <tr class="main-row" :class="{ 'is-expanded': expandedRow === item.id }">
                <td class="name-cell" @click="expandedRow = expandedRow === item.id ? null : item.id">
                  <div class="logo-wrapper">
                <img v-if="getLogo(item.nome)" :src="getLogo(item.nome)" class="logo-img" />
                <i v-else class="fallback-oil-icon">🛢️</i>
                </div>
                  <span class="item-nome">{{ item.nome }}</span>
                  <i class="arrow-icon" :class="{ 'up': expandedRow === item.id }"></i>
                </td>
                <td v-for="ano in anosExibicao" :key="ano">
                    <span v-if="item.totais[ano]" class="val-badge" @click.stop="openProductModal(item.itens, ano, item.nome)">
                    {{ item.totais[ano] }}
                    </span>
                    <span v-else>0</span>
                </td>
              </tr>

              <template v-if="expandedRow === item.id">
                <tr v-for="(subData, subNome) in item.itens" :key="subNome" class="sub-row">
                  <td class="sub-name-cell">
                    <div class="logo-wrapper">
                    <img v-if="getLogo(subNome)" :src="getLogo(subNome)" class="logo-sub-img" />
                    <i v-else class="fallback-oil-icon sub">🛢️</i>
                    </div>
                    <span>{{ subNome === 'nan' ? (filtroPrincipal === 'Indústria' ? 'Sem Montadora' : 'Sem Indústria') : subNome }}</span>
                  </td>
                  <td v-for="ano in anosExibicao" :key="ano">
                        <span v-if="subData.totais[ano]" class="val-badge" @click.stop="openProductModal(subData.produtos, ano, subNome)">
                        {{ subData.totais[ano] }}
                        </span>
                        <span v-else>0</span>
                    </td>
                </tr>
              </template>
            </template>
          </tbody>
          <tbody v-else>
            <tr><td colspan="6" style="padding: 40px; text-align: center; color: #888;">Buscando dados da API...</td></tr>
          </tbody>
        </table>

        <div class="pagination-footer" v-if="totalPages > 1">
          <button :disabled="currentPage === 1" @click="currentPage--">‹ Anterior</button>
          <span class="page-info">Página <strong>{{ currentPage }}</strong> de {{ totalPages }}</span>
          <button :disabled="currentPage === totalPages" @click="currentPage++">Próxima ›</button>
        </div>
      </section>

      <!-- SIDEBAR NEWS -->
      <aside class="lubx-news">
        <div class="news-header">
          <img src="/Lubx_White.png" alt="LUBX" class="news-logo-original" />
          <span>News Mercado Automotivo</span>
        </div>
        
        <div class="news-card-dark">
          <div class="sidebar-tabs">
            <button :class="{ active: activeSidebarTab === 'homologacoes' }" @click="activeSidebarTab = 'homologacoes'">🔥 Novas homologações</button>
            <button :class="{ active: activeSidebarTab === 'indicadores' }" @click="activeSidebarTab = 'indicadores'">📊 Indicadores</button>
          </div>

          <div v-if="activeSidebarTab === 'homologacoes'">
            <h4 class="sidebar-h4">Novos produtos homologados</h4>
            
            <div v-for="(prod, i) in newsHomologacoes" :key="i" class="homolog-card-v2">
              <div class="card-date-v2">Registrado em {{ prod.homologado_em }}</div>
              <div class="card-title-v2">{{ prod.produto }}</div>
              <div class="card-grid">
                <div class="grid-item">
                  <i class="icon-gray">🏢</i> <span class="label-gray">Fabricante</span>
                  <strong class="val-white">{{ prod.fabricante }}</strong>
                </div>
                <div class="grid-item">
                  <i class="icon-gray">🛡️</i> <span class="label-gray">Detentor</span>
                  <div class="detentor-badge">
                    <span class="dot" :class="prod.montadora !== 'nan' ? 'yellow' : 'green'"></span>
                    <strong class="val-white">{{ prod.montadora !== 'nan' ? 'Montadora' : 'Indústria' }}</strong>
                  </div>
                </div>
                <div class="grid-item">
                  <i class="icon-gray">📝</i> <span class="label-gray">Especificação</span>
                  <strong class="val-white">{{ prod.especificacoes }}</strong>
                </div>
                <!-- CAMPO RESTAURADO ABAIXO -->
                <div class="grid-item">
                  <i class="icon-gray">📅</i> <span class="label-gray">Homologado em</span>
                  <strong class="val-white">{{ prod.homologado_em }}</strong>
                </div>
              </div>
            </div>
          </div>

          <div v-if="activeSidebarTab === 'indicadores'" class="indicators-view">
             <div class="indicador-header-lumen">
               <span class="lumen-tag">📊 POSTO EM DIA</span>
               <span class="lumen-date">Quinta · 23 abr 2026</span>
             </div>
             <div class="ind-line"><span>USD/BRL</span> <strong>R$ {{ indicadoresFinanceiros.usd }}</strong></div>
             <div class="ind-line"><span>EUR/BRL</span> <strong>R$ {{ indicadoresFinanceiros.eur }}</strong></div>
          </div>
        </div>
      </aside>
    </div>

    <!-- MODAL DE PRODUTOS -->
    <div v-if="selectedCell" class="product-modal-overlay">
      <div class="product-modal" @click.stop>
        <div class="modal-label">Produtos de {{ selectedCell.nome }} em {{ selectedCell.ano }}</div>
        <table class="modal-prod-table">
          <thead>
            <tr><th>Produto</th><th>Viscosidade</th><th>Finalidade</th></tr>
          </thead>
          <tbody>
            <tr v-for="(p, i) in selectedCell.produtos" :key="i">
              <td>{{ p.produto }}</td>
              <td><span class="p-tag">{{ p.viscosidade || 'N.A' }}</span></td>
              <td><span class="p-tag">{{ p.finalidade || 'Motor' }}</span></td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<style scoped>
.anp-dashboard { background: #f4f4f4; min-height: 100vh; padding: 20px; font-family: 'Segoe UI', sans-serif; color: #333; }
.top-header { display: flex; justify-content: space-between; align-items: center; background: white; padding: 12px 25px; border-radius: 12px; box-shadow: 0 2px 4px rgba(0,0,0,0.05); }
.highlight { color: #f58220; font-weight: 700; }
.search-bar { background: #f9f9f9; border: 1px solid #e0e0e0; border-radius: 8px; padding: 8px 15px; display: flex; align-items: center; width: 300px; }
.search-bar input { border: none; background: transparent; outline: none; margin-left: 10px; width: 100%; }

.filters-row { display: flex; gap: 15px; margin: 15px 0; }
.custom-select { background: white; padding: 8px 15px; border-radius: 8px; border: 1px solid #ddd; cursor: pointer; position: relative; min-width: 220px; font-size: 13px; }
.select-dropdown { position: absolute; top: 100%; left: 0; background: white; width: 100%; border: 1px solid #ddd; border-radius: 8px; z-index: 200; padding: 5px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
.option-label { display: flex; align-items: center; gap: 10px; padding: 8px; cursor: pointer; border-radius: 6px; }
.option-label:hover { background: #f5f5f5; }

.periodo-modal { position: absolute; top: 105%; left: 0; background: white; width: 250px; padding: 15px; border-radius: 10px; box-shadow: 0 10px 30px rgba(0,0,0,0.1); z-index: 201; border: 1px solid #eee; }
.range-inputs { display: flex; gap: 10px; margin-bottom: 10px; }
.range-inputs label { display: block; font-size: 10px; color: #888; margin-bottom: 3px; }
.range-inputs input { width: 100%; padding: 5px; border: 1px solid #ddd; border-radius: 5px; }
.btn-apply { width: 100%; background: #1a1a1a; color: white; border: none; padding: 8px; border-radius: 6px; cursor: pointer; font-weight: bold; }

.main-content { display: grid; grid-template-columns: 1fr 350px; gap: 20px; }
.table-container { background: white; border-radius: 12px; padding: 15px; display: flex; flex-direction: column; overflow-x: auto; }
.table-header-info { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; font-size: 12px; color: #888; }
.btn-export { background: #f58220; color: white; border: none; padding: 6px 12px; border-radius: 6px; font-weight: bold; cursor: pointer; }

.anp-table { width: 100%; border-collapse: collapse; flex: 1; }
/* COMPACTAÇÃO MÁXIMA DA TABELA */
.anp-table th { padding: 8px; color: #8C6239; font-size: 12px; border-bottom: 2px solid #f0f0f0; }
.anp-table td { padding: 4px 8px; text-align: center; border-bottom: 1px solid #f9f9f9; font-size: 12px; height: 35px; min-width: 60px; }
.text-left { text-align: left !important; }

.name-cell { display: flex; align-items: center; gap: 8px; text-align: left !important; font-weight: 600; cursor: pointer; }
.logo-img { width: 20px; height: 20px; object-fit: contain; }
.arrow-icon { border: solid #999; border-width: 0 1.5px 1.5px 0; display: inline-block; padding: 2px; transform: rotate(45deg); transition: 0.3s; margin-left: auto; }
.arrow-icon.up { transform: rotate(-135deg); }

.sub-row { background: #fcfcfc; }
.sub-row td { padding: 3px 8px; height: 30px; } /* ESPAÇAMENTO MÍNIMO EM MONTADORA */
.sub-name-cell { display: flex; align-items: center; gap: 8px; padding-left: 35px !important; text-align: left !important; color: #777; font-size: 11px; }
.logo-sub-img { width: 14px; height: 14px; filter: grayscale(1); }
.val-badge { background: #fff3e6; color: #f58220; padding: 1px 6px; border-radius: 4px; font-weight: bold; cursor: pointer; border: 1px solid #ffe4cc; font-size: 11px; }

.pagination-footer { display: flex; justify-content: center; align-items: center; gap: 15px; margin-top: 15px; padding-top: 10px; border-top: 1px solid #eee; }
.pagination-footer button { background: #fff; border: 1px solid #ddd; padding: 4px 10px; border-radius: 6px; cursor: pointer; font-size: 11px; }

.lubx-news { background: #1a1a1a; border-radius: 15px; padding: 15px; color: white; }
.news-header { display: flex; align-items: center; margin-bottom: 15px; font-size: 10px; color: #777; }
.news-logo-original { height: 14px; max-width: 90px; object-fit: contain; margin-right: 8px; }

.sidebar-tabs { display: flex; background: #262626; padding: 3px; border-radius: 8px; margin-bottom: 15px; }
.sidebar-tabs button { flex: 1; border: none; background: transparent; color: #666; padding: 6px; font-size: 10px; font-weight: bold; cursor: pointer; border-radius: 6px; }
.sidebar-tabs button.active { background: #3d2b1a; color: #f58220; }

.sidebar-h4 { font-size: 14px; margin-bottom: 10px; }
.homolog-card-v2 { background: #222; border-radius: 10px; padding: 10px; margin-bottom: 12px; border-left: 3px solid #8C6239; }
.card-date-v2 { font-size: 9px; color: #f58220; background: #3d2b1a; padding: 2px 8px; border-radius: 20px; display: inline-block; margin-bottom: 8px; }
.card-title-v2 { font-weight: bold; font-size: 12px; color: #fff; margin-bottom: 10px; }
.card-grid { display: flex; flex-direction: column; gap: 6px; }
.grid-item { display: grid; grid-template-columns: 20px 80px 1fr; align-items: center; font-size: 10px; }
.icon-gray { opacity: 0.4; font-style: normal; }
.label-gray { color: #666; }
.val-white { color: #ddd; text-align: right; }
.detentor-badge { display: flex; align-items: center; gap: 4px; justify-self: end; background: #2a2a2a; padding: 2px 6px; border-radius: 4px; }
.dot { width: 5px; height: 5px; border-radius: 50%; }
.dot.green { background: #00ff88; }
.dot.yellow { background: #ffcc00; }

.indicators-view { font-size: 11px; }
.indicador-header-lumen { display: flex; justify-content: space-between; margin-bottom: 12px; }
.lumen-tag { font-weight: bold; color: #f58220; }
.ind-line { display: flex; justify-content: space-between; border-bottom: 1px solid #222; padding: 5px 0; color: #888; }
.ind-line strong { color: #fff; }

.product-modal-overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.6); display: flex; align-items: center; justify-content: center; z-index: 1000; }
.product-modal { background: white; width: 600px; border-radius: 12px; padding: 20px; box-shadow: 0 20px 50px rgba(0,0,0,0.3); }
.modal-label { font-size: 13px; font-weight: bold; color: #f58220; margin-bottom: 15px; border-bottom: 2px solid #fff3e6; padding-bottom: 8px; }
.modal-prod-table { width: 100%; border-collapse: collapse; text-align: left; }
.modal-prod-table th { font-size: 10px; color: #999; text-transform: uppercase; padding: 8px; border-bottom: 1px solid #eee; }
.modal-prod-table td { padding: 8px; font-size: 11px; border-bottom: 1px solid #f9f9f9; }
.p-tag { background: #f5f5f5; border: 1px solid #ddd; padding: 2px 6px; border-radius: 4px; font-size: 9px; color: #777; }

/* Adicione estas classes ao final do seu <style> */
.logo-wrapper {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
}

.fallback-oil-icon {
  font-style: normal;
  font-size: 14px;
  color: #f58220; /* Cor laranja para combinar com seu dashboard */
}

.fallback-oil-icon.sub {
  font-size: 11px;
  opacity: 0.6;
}

/* Ajuste a margem da célula de nome para acomodar o wrapper */
.name-cell, .sub-name-cell {
  gap: 10px !important;
}

</style>
