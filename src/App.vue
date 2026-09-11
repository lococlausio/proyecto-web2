<template>
  <div class="app-container">
    <!-- 1. VISTA DE LOGIN (si no hay usuario autenticado) -->
    <Login v-if="!currentUser" @login-success="onLoginSuccess" />

    <!-- 2. VISTA PRINCIPAL DEL SISTEMA (usuario autenticado) -->
    <div v-else class="main-content">
      <!-- Barra superior con datos de sesión y botón de desconexión -->
      <header class="user-header">
        <div>
          <h2>Emprendedores Locales de Ñuble</h2>
          <p class="session-info">
            Usuario: <strong>{{ currentUser.email }}</strong> | 
            Estado: <span :class="['badge-role', currentUser.rol]">{{ currentUser.rol === 'admin' ? 'Admin conectado' : 'Viewer conectado' }}</span>
          </p>
        </div>
        <button class="btn-logout" @click="handleLogout">Cerrar Sesión</button>
      </header>

      <!-- SECCIÓN EXCLUSIVA PARA ADMIN: Formulario Crear / Editar -->
      <section v-if="currentUser.rol === 'admin'" class="card form-section">
        <h3>{{ editingId ? 'Editar Emprendedor' : 'Registrar Nuevo Emprendedor' }}</h3>
        <form @submit.prevent="handleSubmit">
          <div class="input-grid">
            <input 
              v-model="formData.nombre" 
              type="text" 
              placeholder="Nombre del emprendimiento" 
              required 
            />
            <input 
              v-model="formData.rubro" 
              type="text" 
              placeholder="Rubro (ej: Gastronomía, Tejidos)" 
              required 
            />
            <input 
              v-model="formData.ubicacion" 
              type="text" 
              placeholder="Ubicación (ej: San Carlos, Chillán)" 
            />
          </div>
          <div class="form-actions">
            <button type="submit" class="btn-primary">
              {{ editingId ? 'Actualizar' : 'Agregar' }}
            </button>
            <button 
              v-if="editingId" 
              type="button" 
              class="btn-secondary" 
              @click="cancelEdit"
            >
              Cancelar Edición
            </button>
          </div>
        </form>
      </section>

      <!-- MENSAJE DE RESTRICCIÓN PARA VIEWER -->
      <div v-else class="viewer-notice">
        <p>ℹ️ Tienes permisos de solo lectura (<strong>Viewer</strong>). Solo los administradores pueden registrar, modificar o eliminar emprendedores.</p>
      </div>

      <!-- LISTA DE REGISTROS -->
      <section class="card list-section">
        <h3>Lista de Emprendedores</h3>
        <p v-if="emprendedores.length === 0" class="empty-state">No hay registros disponibles.</p>
        
        <table v-else class="data-table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Nombre</th>
              <th>Rubro</th>
              <th>Ubicación</th>
              <!-- Columna de acciones solo visible para rol admin -->
              <th v-if="currentUser.rol === 'admin'">Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in emprendedores" :key="item.id">
              <td>{{ item.id }}</td>
              <td><strong>{{ item.nombre }}</strong></td>
              <td>{{ item.rubro }}</td>
              <td>{{ item.ubicacion || 'No informada' }}</td>
              <!-- Botones Editar y Eliminar solo visibles para admin -->
              <td v-if="currentUser.rol === 'admin'" class="table-actions">
                <button class="btn-warning" @click="startEdit(item)">Editar</button>
                <button class="btn-danger" @click="handleDelete(item.id)">Eliminar</button>
              </td>
            </tr>
          </tbody>
        </table>
      </section>
    </div>
  </div>
</template>

<script>
import Login from './components/Login.vue';

export default {
  name: 'App',
  components: { Login },
  data() {
    return {
      currentUser: null,
      token: '',
      emprendedores: [],
      formData: {
        nombre: '',
        rubro: '',
        ubicacion: ''
      },
      editingId: null
    };
  },
  mounted() {
    // Al cargar la app, revisar si hay sesión previa guardada en localStorage
    const savedToken = localStorage.getItem('token');
    const savedUser = localStorage.getItem('user');

    if (savedToken && savedUser) {
      this.token = savedToken;
      this.currentUser = JSON.parse(savedUser);
      this.fetchEmprendedores();
    }
  },
  methods: {
    onLoginSuccess(usuario) {
      this.token = localStorage.getItem('token');
      this.currentUser = usuario;
      this.fetchEmprendedores();
    },

    handleLogout() {
      // Limpiar datos locales y restablecer estado
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      this.currentUser = null;
      this.token = '';
      this.emprendedores = [];
      this.cancelEdit();
    },

    async fetchEmprendedores() {
      try {
        const res = await fetch('http://localhost:3000/emprendedores');
        if (res.ok) {
          this.emprendedores = await res.json();
        }
      } catch (err) {
        console.error('Error al cargar emprendedores:', err);
      }
    },

    async handleSubmit() {
      try {
        const isEditing = Boolean(this.editingId);
        const url = isEditing
          ? `http://localhost:3000/emprendedores/${this.editingId}`
          : 'http://localhost:3000/emprendedores';
        
        const method = isEditing ? 'PUT' : 'POST';

        const res = await fetch(url, {
          method,
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${this.token}`
          },
          body: JSON.stringify(this.formData)
        });

        const data = await res.json();

        if (!res.ok) {
          alert(data.error || 'Ocurrió un error en la operación');
          return;
        }

        // Limpiar formulario y recargar datos
        this.cancelEdit();
        this.fetchEmprendedores();
      } catch (err) {
        console.error('Error al guardar registro:', err);
      }
    },

    startEdit(item) {
      this.editingId = item.id;
      this.formData = {
        nombre: item.nombre,
        rubro: item.rubro,
        ubicacion: item.ubicacion || ''
      };
      window.scrollTo({ top: 0, behavior: 'smooth' });
    },

    cancelEdit() {
      this.editingId = null;
      this.formData = { nombre: '', rubro: '', ubicacion: '' };
    },

    async handleDelete(id) {
      if (!confirm('¿Seguro que deseas eliminar este registro?')) return;

      try {
        const res = await fetch(`http://localhost:3000/emprendedores/${id}`, {
          method: 'DELETE',
          headers: {
            'Authorization': `Bearer ${this.token}`
          }
        });

        const data = await res.json();

        if (!res.ok) {
          alert(data.error || 'No fue posible eliminar');
          return;
        }

        this.fetchEmprendedores();
      } catch (err) {
        console.error('Error al eliminar registro:', err);
      }
    }
  }
};
</script>

<style scoped>
.app-container {
  max-width: 900px;
  margin: 0 auto;
  padding: 1.5rem;
  font-family: Arial, sans-serif;
  color: #2c3e50;
}
.user-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 2px solid #eceff1;
  padding-bottom: 1rem;
  margin-bottom: 1.5rem;
}
.session-info {
  margin: 0.3rem 0 0 0;
  color: #555;
  font-size: 0.95rem;
}
.badge-role {
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
  font-size: 0.85rem;
  font-weight: bold;
}
.badge-role.admin {
  background-color: #e8f5e9;
  color: #2e7d32;
}
.badge-role.viewer {
  background-color: #e3f2fd;
  color: #1565c0;
}
.btn-logout {
  background-color: #d32f2f;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}
.card {
  background: white;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
  margin-bottom: 1.5rem;
}
.input-grid {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 0.8rem;
  margin-bottom: 1rem;
}
input {
  padding: 0.6rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 0.95rem;
}
.form-actions {
  display: flex;
  gap: 0.5rem;
}
.btn-primary {
  background-color: #2e7d32;
  color: white;
  border: none;
  padding: 0.6rem 1.2rem;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}
.btn-secondary {
  background-color: #757575;
  color: white;
  border: none;
  padding: 0.6rem 1.2rem;
  border-radius: 4px;
  cursor: pointer;
}
.viewer-notice {
  background-color: #e3f2fd;
  color: #0d47a1;
  padding: 0.8rem 1.2rem;
  border-radius: 6px;
  margin-bottom: 1.5rem;
}
.data-table {
  width: 100%;
  border-collapse: collapse;
  text-align: left;
}
.data-table th, .data-table td {
  padding: 0.75rem;
  border-bottom: 1px solid #eee;
}
.data-table th {
  background-color: #f8f9fa;
}
.table-actions {
  display: flex;
  gap: 0.4rem;
}
.btn-warning {
  background-color: #f57c00;
  color: white;
  border: none;
  padding: 0.35rem 0.7rem;
  border-radius: 4px;
  cursor: pointer;
}
.btn-danger {
  background-color: #e53935;
  color: white;
  border: none;
  padding: 0.35rem 0.7rem;
  border-radius: 4px;
  cursor: pointer;
}
.empty-state {
  color: #777;
  font-style: italic;
}
</style>