<script setup>
import { ref, onMounted } from 'vue';
import { Menu, User } from 'lucide-vue-next';

defineEmits(['toggle-sidebar']);

const userEmail = ref('');
const userName = ref(''); 

onMounted(() => {
  // 1. Pegamos o objeto user_info que salvamos no login
  const savedUser = localStorage.getItem('user_info');

  if (savedUser) {
    try {
      const user = JSON.parse(savedUser);
      
      // 2. Preenchemos as variáveis reativas com os dados do banco
      userName.value = user.nome || 'Usuário';
      userEmail.value = user.email || '';
      
    } catch (error) {
      console.error('Erro ao processar dados do usuário:', error);
    }
  }
});
</script>

<template>
  <header class="navbar navbar-expand bg-white border-bottom sticky-top px-3 py-3">
    
    <!-- Botão Hambúrguer -->
    <button 
      class="btn btn-light d-md-none me-3 border" 
      @click="$emit('toggle-sidebar')"
    >
      <Menu :size="20" />
    </button>

    <div class="d-flex flex-column">
      <h5 class="fw-bold m-0 text-dark" style="font-size: 1.1rem;">BUSINESS INTELLIGENCE - TOTALENERGIES</h5>
      <small class="text-muted d-none d-sm-block">Sistema de Gestão Automotiva da LubConsulta</small>
    </div>

    <div class="ms-auto d-flex align-items-center gap-3">
      <div class="text-end d-none d-md-block">
        <div class="fw-bold text-dark text-uppercase" style="font-size: 0.9rem;">{{ userName }}</div>
        <div class="text-muted" style="font-size: 0.8rem;">{{ userEmail }}</div>
      </div>
      
      <!-- Avatar Padrão (Mostra a primeira letra do nome) -->
      <div class="avatar-circle d-flex align-items-center justify-content-center bg-orange-subtle text-primary fw-bold border border-warning">
        {{ userName.charAt(0).toUpperCase() }}
      </div>

    </div>
  </header>
</template>

<style scoped>
.bg-orange-subtle { background-color: #fff7ed !important; }
.text-primary { color: #f97316 !important; }
.avatar-circle {
  width: 42px; 
  height: 42px; 
  border-radius: 50%;
  font-size: 1.2rem;
}
</style>
