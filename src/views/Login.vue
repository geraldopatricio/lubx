<script setup>
import { useRouter } from 'vue-router';
import { ref } from 'vue';
import { Eye, EyeOff, AlertTriangle, CheckCircle } from 'lucide-vue-next';

const router = useRouter();

// === ESTADOS DE LOGIN ===
const email = ref('');
const password = ref('');
const showPassword = ref(false);
const isLoading = ref(false);

// === SISTEMA DE TOAST ===
const toastMessage = ref('');
const toastType = ref('error'); 

const showToast = (msg, type = 'error') => {
  toastMessage.value = msg;
  toastType.value = type;
  setTimeout(() => { toastMessage.value = ''; }, 5000);
};

const togglePassword = () => showPassword.value = !showPassword.value;

// === LÓGICA DE LOGIN CONECTADA AO BACKEND ===
const handleLogin = async () => {
  if (!email.value || !password.value) {
    showToast('Preencha todos os campos', 'error');
    return;
  }

  toastMessage.value = '';
  isLoading.value = true;

  try {
    const response = await fetch('https://lubx-api.lubconsulta.com.br/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        email: email.value,
        senha: password.value
      })
    });

    const data = await response.json();

    if (!response.ok) {
      // Aqui ele vai exibir "E-mail ou senha incorretos" ou "Seu acesso está bloqueado"
      throw new Error(data.message || 'Erro ao realizar login');
    }

    // SUCESSO
    localStorage.setItem('authToken', data.token);
    localStorage.setItem('user_info', JSON.stringify(data.user));
    
    showToast(`Bem-vindo, ${data.user.nome}!`, 'success');
    
    setTimeout(() => { router.push('/'); }, 800);

  } catch (error) {
    showToast(error.message, 'error');
  } finally {
    isLoading.value = false;
  }
};

const openForgotModal = () => { showToast('Funcionalidade em desenvolvimento.', 'success'); };
const openModal = () => { showToast('O cadastro deve ser solicitado ao administrador.', 'error'); };
</script>

<template>
  <div class="d-flex align-items-center justify-content-center min-vh-100 bg-light position-relative overflow-hidden">
    
    <!-- TOAST -->
    <Transition name="toast">
      <div v-if="toastMessage" :class="['custom-toast', toastType === 'success' ? 'toast-success' : 'toast-error']">
        <CheckCircle v-if="toastType === 'success'" class="me-2" :size="20" />
        <AlertTriangle v-else class="me-2" :size="20" />
        <span>{{ toastMessage }}</span>
      </div>
    </Transition>
    
    <div class="card border-0 shadow-lg p-4" style="max-width: 400px; width: 100%;">
      <div class="card-body">
        <div class="text-center mb-4">
          <img src="/Lubx_01.png" alt="Logo" style="width: 200px; height: 52px;" class="mb-3">
          <h4 class="fw-bold text-dark">Área de Acesso</h4>
          <p class="text-muted small">BUSINESS INTELLIGENCE - TOTALENERGIES</p>
        </div>

        <form @submit.prevent="handleLogin">
          <div class="form-floating mb-3 custom-floating">
            <input type="email" class="form-control" id="emailInput" placeholder="name@example.com" v-model="email" required :disabled="isLoading">
            <label for="emailInput">E-mail</label>
          </div>

          <div class="form-floating mb-3 position-relative custom-floating">
            <input :type="showPassword ? 'text' : 'password'" class="form-control pe-5" id="passwordInput" placeholder="Senha" v-model="password" required :disabled="isLoading">
            <label for="passwordInput">Senha</label>
            <button type="button" class="btn btn-link position-absolute text-muted p-0" style="top: 50%; right: 15px; transform: translateY(-50%); z-index: 10;" @click="togglePassword" :disabled="isLoading">
              <Eye v-if="!showPassword" :size="20" />
              <EyeOff v-else :size="20" />
            </button>
          </div>

          <div class="d-flex justify-content-between align-items-center mb-4">
            <div class="form-check">
              <input class="form-check-input" type="checkbox" id="remember" :disabled="isLoading">
              <label class="form-check-label small text-muted" for="remember">Lembrar-me</label>
            </div>
          </div>

          <button type="submit" class="btn btn-primary w-100 py-2 fw-bold text-white shadow-sm hover-effect d-flex justify-content-center align-items-center gap-2" :disabled="isLoading">
            <span v-if="isLoading" class="spinner-border spinner-border-sm" role="status"></span>
            {{ isLoading ? 'Validando...' : 'Entrar no Sistema' }}
          </button>
        </form>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Estilos mantidos... */
.custom-toast { position: fixed; top: 50px; left: 50%; transform: translateX(-50%); color: white; padding: 12px 24px; border-radius: 50px; box-shadow: 0 10px 25px rgba(0,0,0, 0.2); z-index: 9999; display: flex; align-items: center; font-weight: 600; font-size: 0.9rem; pointer-events: none; }
.toast-error { background-color: #ef4444; }
.toast-success { background-color: #22c55e; }
.custom-floating > .form-control { border-radius: 8px; border: 1px solid #dee2e6; height: 50px; }
.custom-floating > label { padding: 0.7rem 0.75rem; color: #6c757d; }
.hover-effect:hover { filter: brightness(1.1); transition: 0.3s; }
</style>
