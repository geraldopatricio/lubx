<script setup>
import { ref, onMounted, computed } from 'vue';
import { 
  Plus, Edit2, Trash2, Search, RefreshCw, X, Lock, Unlock, Eye, UserX, UserCheck
} from 'lucide-vue-next';

const API_URL = 'http://localhost:3000/auth/usuarios';

const users = ref([]);
const searchQuery = ref('');
const isLoading = ref(false);
const isSaving = ref(false);
const showModal = ref(false);
const isEditing = ref(false);

const form = ref({
  id: null,
  nome: '',
  email: '',
  senha: '',
  telefone: '',
  perfil: 'user',
  status: 'Ativo' // Valor inicial
});

const toast = ref({ show: false, msg: '', type: 'success' });

const fetchUsers = async () => {
  isLoading.value = true;
  try {
    const res = await fetch(API_URL);
    users.value = await res.json();
  } catch (e) { showToast('Erro ao carregar', 'error'); }
  finally { isLoading.value = false; }
};

const handleSave = async () => {
  isSaving.value = true;
  try {
    const url = isEditing.value ? `${API_URL}/${form.value.id}` : API_URL;
    const method = isEditing.value ? 'PUT' : 'POST';

    const response = await fetch(url, {
      method,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(form.value)
    });

    if (!response.ok) throw new Error('Falha na operação');

    showToast(isEditing.value ? 'Atualizado!' : 'Cadastrado!');
    showModal.value = false;
    fetchUsers();
  } catch (e) { showToast(e.message, 'error'); }
  finally { isSaving.value = false; }
};

const toggleStatus = async (user) => {
  const novoStatus = user.status === 'Ativo' ? 'Bloqueado' : 'Ativo';
  try {
    await fetch(`${API_URL}/${user.id}/status`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ status: novoStatus })
    });
    fetchUsers();
  } catch (e) { showToast('Erro ao mudar status', 'error'); }
};

const confirmDelete = async (id) => {
  if (!confirm('Excluir permanentemente?')) return;
  try {
    await fetch(`${API_URL}/${id}`, { method: 'DELETE' });
    fetchUsers();
    showToast('Removido!', 'success');
  } catch (e) { showToast('Erro ao remover', 'error'); }
};

const openModalCreate = () => {
  isEditing.value = false;
  form.value = { id: null, nome: '', email: '', senha: '', telefone: '', perfil: 'user', status: 'Ativo' };
  showModal.value = true;
};

const openModalEdit = (user) => {
  isEditing.value = true;
  form.value = { ...user, senha: '' };
  showModal.value = true;
};

const showToast = (msg, type = 'success') => {
  toast.value = { show: true, msg, type };
  setTimeout(() => toast.value.show = false, 3000);
};

const filteredUsers = computed(() => {
  return users.value.filter(u => u.nome.toLowerCase().includes(searchQuery.value.toLowerCase()));
});

onMounted(fetchUsers);
</script>

<template>
  <div class="container-fluid p-4">
    <div v-if="toast.show" :class="['custom-toast', toast.type === 'success' ? 'bg-success' : 'bg-danger']">{{ toast.msg }}</div>

    <div class="d-flex justify-content-between mb-4">
      <h4 class="fw-bold">GESTÃO DE ACESSOS</h4>
      <button class="btn btn-primary text-white" @click="openModalCreate"><Plus :size="18"/> Novo Usuário</button>
    </div>

    <div class="card border-0 shadow-sm">
      <div class="table-responsive">
        <table class="table table-hover align-middle mb-0">
          <thead class="bg-light">
            <tr>
              <th class="ps-4">NOME</th>
              <th>E-MAIL</th>
              <th>PERFIL</th>
              <th>STATUS</th>
              <th class="text-end pe-4">AÇÕES</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="user in filteredUsers" :key="user.id">
              <td class="ps-4">{{ user.nome }}</td>
              <td>{{ user.email }}</td>
              <td><span class="badge bg-secondary">{{ user.perfil }}</span></td>
              <td>
                <span :class="['badge', user.status === 'Ativo' ? 'bg-success' : 'bg-danger']">{{ user.status }}</span>
              </td>
              <td class="text-end pe-4">
                <div class="btn-group">
                  <button class="btn btn-sm btn-light text-primary" @click="openModalEdit(user)"><Edit2 :size="16"/></button>
                  <button class="btn btn-sm btn-light" @click="toggleStatus(user)">
                    <UserX v-if="user.status === 'Ativo'" :size="16" class="text-warning"/>
                    <UserCheck v-else :size="16" class="text-success"/>
                  </button>
                  <button class="btn btn-sm btn-light text-danger" @click="confirmDelete(user.id)"><Trash2 :size="16"/></button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- MODAL COM CAMPO STATUS -->
    <div v-if="showModal" class="modal-overlay d-flex align-items-center justify-content-center">
      <div class="overlay-bg" @click="showModal = false"></div>
      <div class="card modal-content p-4" style="max-width: 500px; width: 90%;">
        <h5 class="fw-bold mb-3">{{ isEditing ? 'Editar Usuário' : 'Novo Usuário' }}</h5>
        <form @submit.prevent="handleSave" class="row g-3">
          <div class="col-12">
            <label class="form-label">Nome</label>
            <input type="text" class="form-control" v-model="form.nome" required>
          </div>
          <div class="col-md-6">
            <label class="form-label">E-mail</label>
            <input type="email" class="form-control" v-model="form.email" required>
          </div>
          <div class="col-md-6">
            <label class="form-label">Telefone</label>
            <input type="text" class="form-control" v-model="form.telefone">
          </div>
          <div class="col-md-6">
            <label class="form-label">Perfil</label>
            <select class="form-select" v-model="form.perfil">
              <option value="user">Usuário</option>
              <option value="admin">Administrador</option>
            </select>
          </div>
          <div class="col-md-6">
            <label class="form-label">Status do Acesso</label>
            <select class="form-select" v-model="form.status">
              <option value="Ativo">Ativo</option>
              <option value="Bloqueado">Bloqueado</option>
            </select>
          </div>
          <div class="col-12" v-if="!isEditing || form.senha">
            <label class="form-label">Senha</label>
            <input type="password" class="form-control" v-model="form.senha" :required="!isEditing">
          </div>
          <div class="col-12 d-flex gap-2 justify-content-end mt-4">
            <button type="button" class="btn btn-light" @click="showModal = false">Cancelar</button>
            <button type="submit" class="btn btn-primary text-white" :disabled="isSaving">Salvar</button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<style scoped>
.modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; z-index: 1050; }
.overlay-bg { position: absolute; width: 100%; height: 100%; background: rgba(0,0,0,0.5); backdrop-filter: blur(4px); }
.modal-content { z-index: 1060; background: white; border-radius: 12px; }
.custom-toast { position: fixed; top: 20px; left: 50%; transform: translateX(-50%); color: white; padding: 10px 20px; border-radius: 50px; z-index: 2000; }
</style>
