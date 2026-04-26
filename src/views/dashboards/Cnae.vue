<script setup>
import { ref } from 'vue';
import { Globe, LayoutDashboard, ExternalLink } from 'lucide-vue-next';

// Estado para controle de carregamento, caso queira futuramente
const isLoading = ref(true);

const handleLoad = () => {
  isLoading.value = false;
};
</script>

<template>
  <div class="dashboard-container">
    <!-- CABEÇALHO -->
    <div class="p-4 bg-white shadow-sm d-flex justify-content-between align-items-center">
      <div class="d-flex align-items-center gap-3">
        <div class="icon-circle-bg">
          <LayoutDashboard class="text-orange" :size="24"/>
        </div>
        <div>
          <h4 class="fw-bold m-0 text-dark">BI Oportunidades</h4>
          <p class="text-muted small mb-0">Análise Estratégica - TotalEnergies</p>
        </div>
      </div>
    </div>

    <!-- ÁREA DO POWER BI -->
    <div class="iframe-wrapper p-3">
      <div class="card border-0 shadow-sm rounded-4 overflow-hidden bg-white h-100 position-relative">
        
        <!-- Overlay de carregamento -->
        <div v-if="isLoading" class="loader-overlay">
          <div class="spinner-border text-orange" role="status"></div>
          <span class="mt-2 fw-bold text-muted">Carregando Relatório...</span>
        </div>

        <!-- ADICIONADO: class="powerbi-iframe" e @load="handleLoad" -->
        <iframe 
          class="powerbi-iframe"
          title="Bi Cnaes LubConsulta" 
          src="https://app.powerbi.com/view?r=eyJrIjoiZjg3MjgzOWYtNzBjNi00NTlkLTkzM2MtYTAwNmRlMWM0NDk1IiwidCI6ImEzYTY5ODI1LTA2YTMtNDU0Ny1iZGZkLTBlYWI3MDJmMTcyNiJ9" 
          frameborder="0" 
          allowFullScreen="true"
          @load="handleLoad"
        ></iframe>

      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import { LayoutDashboard } from 'lucide-vue-next';

const isLoading = ref(true);

const handleLoad = () => {
  console.log("Iframe carregado!");
  isLoading.value = false;
};
</script>

<style scoped>
/* Garante que o container ocupe a tela toda */
.dashboard-container {
  background-color: #f1f5f9;
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.iframe-wrapper {
  flex: 1; /* Faz o wrapper ocupar o resto da tela */
  display: flex;
}

/* O iframe deve ocupar 100% do card */
.powerbi-iframe {
  width: 100%;
  height: 100%;
  border: none;
}

.loader-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: white;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 10;
}

.text-orange { color: #e97332 !important; }
.icon-circle-bg { 
  width: 45px; 
  height: 45px; 
  border-radius: 50%; 
  display: flex; 
  align-items: center; 
  justify-content: center; 
  background: #fff1eb; 
}
</style>
