<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import axios from 'axios';
import { 
  Search, FileText, ChevronDown, ChevronUp, Layers, Info, CheckCircle, 
  Package, AlertCircle, X, Tag, LayoutGrid, List
} from 'lucide-vue-next';

// --- CONFIGURAÇÕES ---
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3000';
const S3_IMG_BUCKET_URL = 'https://lubconsulta-imagem-lubrificante-prod.s3.sa-east-1.amazonaws.com';
const S3_PDF_BUCKET_URL = 'https://lubconsulta-manual-lubrificante-prod.s3.sa-east-1.amazonaws.com';

const IBP_BRANDS = ['LUBRAX', 'IPIRANGA', 'TEXACO', 'MOBIL', 'SHELL', 'PETRONAS', 'CASTROL', 'TOTALENERGIES', 'YPF'];

// --- ESTADO ---
const searchQuery = ref('');
const showSuggestions = ref(false);
const loading = ref(false);
const errorMsg = ref('');
const searchContainer = ref(null); // Ref para detectar clique fora

const suggestionsList = ref([]);
const selectedProduct = ref(null);   
const processedSimilars = ref([]); 
const viewMode = ref('cards'); // 'cards' ou 'list'

const showIBP = ref(true);
const showDiverse = ref(true);
const showPdfModal = ref(false);
const currentPdfUrl = ref('');

// --- HELPERS ---
const slugify = (text) => text ? text.toLowerCase().trim().replace(/[\s/]+/g, '-') : '';
const getProductImage = (name) => `${S3_IMG_BUCKET_URL}/${slugify(name)}.webp`;
const handleImageError = (e) => { e.target.src = 'https://placehold.co/100x160/f1f5f9/94a3b8?text=Sem+Imagem'; };

const formatDisplayValue = (value) => {
    if (!value || value === '' || value === 'null') return '-';
    if (Array.isArray(value)) return value.join(', ') || '-';
    return String(value);
};

// Verifica se é óleo de moto
const isMotoProduct = (product) => {
    const tipos = Array.isArray(product?.tipo_veiculo) ? product.tipo_veiculo.map(t => t.toUpperCase()) : [];
    return tipos.some(t => ['MOTO', 'MOTOCICLETA'].includes(t));
};

// --- FECHAR FILTRO (CLICK FORA E ESC) ---
const handleClickOutside = (event) => {
  if (searchContainer.value && !searchContainer.value.contains(event.target)) {
    showSuggestions.value = false;
  }
};

const handleEscKey = (event) => {
  if (event.key === 'Escape') showSuggestions.value = false;
};

onMounted(() => {
  document.addEventListener('mousedown', handleClickOutside);
  document.addEventListener('keydown', handleEscKey);
});

onUnmounted(() => {
  document.removeEventListener('mousedown', handleClickOutside);
  document.removeEventListener('keydown', handleEscKey);
});

// --- LÓGICA DE SIMILARIDADE ---
const analyzeProduct = (base, target) => {
  const tags = [];
  let score = 0;

  // Regra de Ouro: Viscosidade idêntica costuma ser o primeiro filtro
  if (base.viscosidade === target.viscosidade) score += 20;

  // Regra VW e Normas: Se o base tem norma e o target também tem a mesma norma
  if (base.oem && target.oem) {
      const baseNorms = String(base.oem).toUpperCase();
      const targetNorms = String(target.oem).toUpperCase();
      
      // Se for VW, a norma é mandatória para similaridade alta
      if (baseNorms.includes('VW')) {
          const vwNorms = baseNorms.match(/VW\s\d{3}\.\d{2}/g) || [];
          const hasSameVwNorm = vwNorms.some(norm => targetNorms.includes(norm));
          if (hasSameVwNorm) {
              tags.push({ text: 'NORMA VW ATENDIDA', type: 'success' });
              score += 50;
          } else if (vwNorms.length > 0) {
              tags.push({ text: 'NORMA VW DIVERGENTE', type: 'danger' });
              score -= 30;
          }
      }
  }

  // Comparação API
  const sBase = getApiScore(base.api);
  const sTarget = getApiScore(target.api);
  if (sTarget > sBase) { tags.push({ text: 'API SUPERIOR', type: 'success' }); score += 10; }
  else if (sTarget === sBase) score += 5;

  return { tags, score };
};

const getApiScore = (apiStr) => {
  if (!apiStr) return 0;
  const api = String(apiStr).toUpperCase();
  if (api.includes('SP')) return 10;
  if (api.includes('SN')) return 9;
  if (api.includes('SL')) return 8;
  if (api.includes('CK-4')) return 20;
  return 1;
};

// --- AÇÕES ---
const fetchSuggestions = async (term) => {
  loading.value = true;
  try {
    const response = await axios.get(`${API_URL}/produtos/autocomplete?q=${term}`);
    suggestionsList.value = response.data || [];
  } catch (err) {
    errorMsg.value = "Erro na busca.";
  } finally { loading.value = false; }
};

let debounceTimeout = null;
const handleInput = () => {
  searchQuery.value = searchQuery.value.toUpperCase();
  showSuggestions.value = true;
  if (debounceTimeout) clearTimeout(debounceTimeout);
  if (searchQuery.value.length < 2) { suggestionsList.value = []; return; }
  debounceTimeout = setTimeout(() => fetchSuggestions(searchQuery.value), 400);
};

const selectProduct = async (itemSimple) => {
  searchQuery.value = itemSimple.nome;
  showSuggestions.value = false;
  loading.value = true;
  try {
      const res = await axios.get(`${API_URL}/produto/${encodeURIComponent(itemSimple.nome)}`);
      if (res.data && res.data.produto) {
          selectedProduct.value = res.data.produto;
          processedSimilars.value = (res.data.similares || []).map(sim => ({
              ...sim,
              analysis: analyzeProduct(res.data.produto, sim)
          })).sort((a, b) => b.analysis.score - a.analysis.score);
      }
  } catch (e) { alert("Erro ao carregar produto."); }
  finally { loading.value = false; }
};

const openPdf = (name) => {
  currentPdfUrl.value = `${S3_PDF_BUCKET_URL}/${slugify(name)}.pdf`;
  showPdfModal.value = true;
};

const groupedSimilars = computed(() => {
  const ibp = [];
  const diverse = [];
  processedSimilars.value.forEach(item => {
    const brand = item.marca?.toUpperCase();
    IBP_BRANDS.includes(brand) ? ibp.push(item) : diverse.push(item);
  });
  return { ibp, diverse };
});
</script>

<template>
  <div class="container-fluid p-4 dashboard-bg">
    
    <!-- HEADER -->
    <div class="mb-4 d-flex justify-content-between align-items-center">
      <div>
        <h4 class="fw-bold text-dark m-0 d-flex align-items-center gap-2">
            <Layers class="text-primary"/> BUSCA DE SIMILARIDADE
        </h4>
        <small class="text-muted">Encontre lubrificantes equivalentes baseados em especificações técnicas.</small>
      </div>
      
      <!-- TOGGLE VISUALIZAÇÃO -->
      <div v-if="selectedProduct" class="btn-group shadow-sm">
        <button @click="viewMode = 'cards'" class="btn btn-white border" :class="{'active btn-primary': viewMode === 'cards'}">
          <LayoutGrid :size="18"/>
        </button>
        <button @click="viewMode = 'list'" class="btn btn-white border" :class="{'active btn-primary': viewMode === 'list'}">
          <List :size="18"/>
        </button>
      </div>
    </div>

    <!-- BARRA DE PESQUISA -->
    <div class="card border-0 shadow-sm mb-4" style="z-index: 1000; position: relative;" ref="searchContainer">
      <div class="card-body p-3">
        <div class="position-relative">
          <div class="input-group input-group-lg border rounded overflow-hidden">
            <span class="input-group-text bg-white border-0 ps-3">
              <Search v-if="!loading" class="text-muted" :size="20"/>
              <span v-else class="spinner-border spinner-border-sm text-primary"></span>
            </span>
            <input 
              type="text" class="form-control border-0 shadow-none fw-bold"
              placeholder="Pesquisar produto referência..."
              v-model="searchQuery" @input="handleInput" @focus="showSuggestions = true"
            >
          </div>

          <!-- LISTA AUTOCOMPLETE (AJUSTE 1) -->
          <ul v-if="showSuggestions && suggestionsList.length > 0" class="list-group position-absolute w-100 shadow mt-1 fade-in custom-scroll">
            <li 
              v-for="item in suggestionsList" :key="item.id"
              class="list-group-item list-group-item-action cursor-pointer d-flex align-items-center gap-3 py-3"
              @click="selectProduct(item)"
            >
               <img :src="getProductImage(item.nome)" class="img-mini" @error="handleImageError">
               <div class="flex-grow-1">
                 <div class="fw-bold text-dark">{{ item.nome }}</div>
                 <div class="d-flex gap-2 mt-1 flex-wrap">
                   <span class="badge bg-light text-primary border">SAE {{ item.viscosidade }}</span>
                   <span class="badge bg-light text-secondary border">API {{ item.api }}</span>
                   <span v-if="item.acea" class="badge bg-light text-secondary border">ACEA {{ item.acea }}</span>
                   <span v-if="item.oem" class="badge bg-light text-info border">NORMA: {{ item.oem }}</span>
                 </div>
               </div>
            </li>
          </ul>
        </div>
      </div>
    </div>

    <!-- PRODUTO BASE (AJUSTE 5) -->
    <div v-if="selectedProduct" class="animate-up">
      <div class="card border-primary border-2 shadow mb-4">
        <div class="card-body p-4">
          <div class="row align-items-center">
            <div class="col-md-2 text-center">
               <img :src="getProductImage(selectedProduct.nome)" class="img-fluid drop-shadow main-img" @error="handleImageError">
            </div>
            <div class="col-md-10">
              <div class="d-flex justify-content-between align-items-start">
                <h3 class="fw-bold text-dark">{{ selectedProduct.nome }}</h3>
                <span class="badge bg-primary px-3 py-2">REFERÊNCIA</span>
              </div>
              
              <div class="row g-3 mt-2">
                <div class="col-md-3">
                  <small class="text-muted fw-bold d-block">VISCOSIDADE</small>
                  <span class="fs-5 fw-bold">{{ selectedProduct.viscosidade }}</span>
                </div>
                <div class="col-md-3">
                  <small class="text-muted fw-bold d-block">API</small>
                  <span>{{ formatDisplayValue(selectedProduct.api) }}</span>
                </div>
                <div class="col-md-3">
                  <small class="text-muted fw-bold d-block">ACEA</small>
                  <span>{{ formatDisplayValue(selectedProduct.acea) }}</span>
                </div>
                <!-- JASO Condicional -->
                <div class="col-md-3" v-if="isMotoProduct(selectedProduct)">
                  <small class="text-muted fw-bold d-block">JASO</small>
                  <span class="text-danger fw-bold">{{ formatDisplayValue(selectedProduct.jaso) }}</span>
                </div>
                <div class="col-12 mt-3 p-3 bg-light rounded">
                  <small class="text-muted fw-bold d-block">NORMAS E ESPECIFICAÇÕES (OEM)</small>
                  <span class="text-dark fw-bold small">{{ formatDisplayValue(selectedProduct.oem) }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- RESULTADOS -->
      <div v-for="(list, title) in { 'MARCAS IBP': groupedSimilars.ibp, 'OUTRAS MARCAS': groupedSimilars.diverse }" :key="title">
        <h6 class="fw-bold mt-4 mb-3 d-flex align-items-center gap-2">
            <div class="dot bg-primary"></div> {{ title }} ({{ list.length }})
        </h6>

        <!-- VIEW EM CARDS (AJUSTE 6) -->
        <div v-if="viewMode === 'cards'" class="row g-3">
          <div v-for="prod in list" :key="prod.id" class="col-12 col-sm-6 col-lg-3">
            <div class="card h-100 border-0 shadow-sm product-card">
              <div class="card-body p-3">
                <div class="d-flex justify-content-between">
                    <small class="fw-bold text-primary">{{ prod.marca }}</small>
                    <div class="d-flex flex-column gap-1">
                        <span v-for="tag in prod.analysis.tags" :class="`badge bg-${tag.type}`" style="font-size:0.6rem">{{ tag.text }}</span>
                    </div>
                </div>
                
                <div class="text-center my-3" style="height: 120px;">
                    <img :src="getProductImage(prod.nome)" class="h-100 drop-shadow" @error="handleImageError">
                </div>

                <div class="info-grid">
                    <div class="info-item"><b>SAE:</b> {{ prod.viscosidade }}</div>
                    <div class="info-item"><b>API:</b> {{ formatDisplayValue(prod.api) }}</div>
                    <div class="info-item"><b>ACEA:</b> {{ formatDisplayValue(prod.acea) }}</div>
                    <div class="info-item" v-if="isMotoProduct(prod)"><b>JASO:</b> {{ formatDisplayValue(prod.jaso) }}</div>
                </div>
                
                <div class="mt-2 pt-2 border-top">
                    <div class="text-dark fw-bold small text-truncate">{{ prod.nome }}</div>
                    <div class="text-muted x-small text-truncate mt-1" :title="prod.oem"><b>Normas:</b> {{ formatDisplayValue(prod.oem) }}</div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- VIEW EM LISTA (AJUSTE 3) -->
        <div v-else class="card border-0 shadow-sm overflow-hidden">
            <table class="table table-hover align-middle mb-0">
                <thead class="bg-light">
                    <tr>
                        <th>Produto</th>
                        <th>Viscosidade</th>
                        <th>API</th>
                        <th>ACEA</th>
                        <th>Normas/OEM</th>
                        <th v-if="isMotoProduct(selectedProduct)">JASO</th>
                        <th>Ficha</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="prod in list" :key="prod.id">
                        <td>
                            <div class="d-flex align-items-center gap-2">
                                <img :src="getProductImage(prod.nome)" width="30" @error="handleImageError">
                                <div>
                                    <div class="fw-bold small">{{ prod.nome }}</div>
                                    <small class="text-muted">{{ prod.marca }}</small>
                                </div>
                            </div>
                        </td>
                        <td><span class="badge bg-light text-dark border">{{ prod.viscosidade }}</span></td>
                        <td><small>{{ formatDisplayValue(prod.api) }}</small></td>
                        <td><small>{{ formatDisplayValue(prod.acea) }}</small></td>
                        <td style="max-width: 250px;"><small class="x-small">{{ formatDisplayValue(prod.oem) }}</small></td>
                        <td v-if="isMotoProduct(selectedProduct)">{{ formatDisplayValue(prod.jaso) }}</td>
                        <td>
                            <button @click="openPdf(prod.nome)" class="btn btn-sm btn-outline-primary border-0">
                                <FileText :size="16"/>
                            </button>
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>
      </div>
    </div>

    <!-- MODAL PDF -->
    <div v-if="showPdfModal" class="modal-backdrop fade show" style="z-index: 1050;"></div>
    <div v-if="showPdfModal" class="modal fade show d-block" @click.self="showPdfModal = false" style="z-index: 1055;">
      <div class="modal-dialog modal-xl modal-dialog-centered h-100 my-0">
        <div class="modal-content h-75 shadow-lg">
          <div class="modal-header py-2 bg-dark text-white">
            <h6 class="modal-title">Ficha Técnica</h6>
            <button class="btn btn-sm btn-dark" @click="showPdfModal = false"><X :size="20"/></button>
          </div>
          <iframe :src="currentPdfUrl" class="w-100 h-100 border-0"></iframe>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
.dashboard-bg { background-color: #f1f5f9; min-height: 100vh; font-family: 'Inter', sans-serif; }
.cursor-pointer { cursor: pointer; }
.img-mini { width: 45px; height: 45px; object-fit: contain; }
.main-img { max-height: 180px; }
.drop-shadow { filter: drop-shadow(2px 5px 8px rgba(0,0,0,0.2)); }
.custom-scroll { max-height: 400px; overflow-y: auto; }

.info-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 5px;
    font-size: 0.75rem;
    background: #f8fafc;
    padding: 8px;
    border-radius: 6px;
}

.x-small { font-size: 0.7rem; line-height: 1.1; }

.product-card { 
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
    border: 1px solid transparent !important;
}
.product-card:hover { 
    transform: translateY(-5px); 
    box-shadow: 0 12px 20px -5px rgba(0,0,0,0.1) !important;
    border-color: #0ea5e9 !important;
}

.dot { width: 8px; height: 8px; border-radius: 50%; }
.animate-up { animation: slideUp 0.4s ease-out; }

@keyframes slideUp { 
  from { opacity: 0; transform: translateY(20px); } 
  to { opacity: 1; transform: translateY(0); } 
}

.table th { font-size: 0.8rem; text-transform: uppercase; color: #64748b; font-weight: 700; }
</style>
