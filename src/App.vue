<template>
  <div class="app-container">
    <header class="app-header">
      <h2>Emprendedores Locales de Ñuble</h2>
      <p class="subtitle">Backend NestJS + TypeORM + MySQL</p>
    </header>

    <!-- FORMULARIO CREAR / EDITAR -->
    <section class="card form-section">
      <h3>{{ editingId ? 'Editar Emprendedor' : 'Registrar Nuevo Emprendedor' }}</h3>
      <form @submit.prevent="handleSubmit">
        <div class="input-grid">
          <input 
            v-model="formData.nombre" 
            type="text" 
            placeholder="Nombre (mín. 3 letras)" 
            required 
          />
          <input 
            v-model="formData.comuna" 
            type="text" 
            placeholder="Comuna (ej: Chillán, San Carlos)" 
            required 
          />
          <select v-model="formData.rubro" required>
            <option disabled value="">Seleccione un Rubro</option>
            <option value="Apicultura">Apicultura</option>
            <option value="Lácteos">Lácteos</option>
            <option value="Textiles">Textiles</option>
            <option value="Turismo">Turismo</option>
            <option value="Artesanía">Artesanía</option>
            <option value="Agricultura">Agricultura</option>
          </select>
          <input 
            v-model="formData.contacto" 
            type="text" 
            placeholder="Contacto (email o teléfono)" 
            required 
          />
        </div>

        <div class="full-width">
          <textarea 
            v-model="formData.descripcion" 
            placeholder="Descripción (mínimo 10 caracteres)" 
            rows="2" 
            required
          ></textarea>
        </div>

        <div v-if="errorMessage" class="error-banner">
          {{ errorMessage }}
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

    <!-- BUSCADOR CON FILTROS (GET /emprendedores/buscar) -->
    <section class="card filter-section">
      <h3>Buscar Emprendedores</h3>
      <div class="filter-grid">
        <input 
          v-model="filterComuna" 
          type="text" 
          placeholder="Filtrar por comuna..." 
        />
        <select v-model="filterRubro">
          <option value="">Todos los rubros</option>
          <option value="Apicultura">Apicultura</option>
          <option value="Lácteos">Lácteos</option>
          <option value="Textiles">Textiles</option>
          <option value="Turismo">Turismo</option>
          <option value="Artesanía">Artesanía</option>
          <option value="Agricultura">Agricultura</option>
        </select>
        <button class="btn-search" @click="handleSearch">Buscar</button>
        <button class="btn-secondary" @click="resetSearch">Limpiar</button>
      </div>
    </section>

    <!-- LISTA DE REGISTROS -->
    <section class="card list-section">
      <h3>Catálogo de Emprendimientos</h3>
      <p v-if="emprendedores.length === 0" class="empty-state">No se encontraron registros.</p>
      
      <table v-else class="data-table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Nombre</th>
            <th>Comuna</th>
            <th>Rubro</th>
            <th>Descripción</th>
            <th>Contacto</th>
            <th>Acciones</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in emprendedores" :key="item.id">
            <td>{{ item.id }}</td>
            <td><strong>{{ item.nombre }}</strong></td>
            <td>{{ item.comuna }}</td>
            <td><span class="badge">{{ item.rubro }}</span></td>
            <td>{{ item.descripcion }}</td>
            <td>{{ item.contacto }}</td>
            <td class="table-actions">
              <button class="btn-warning" @click="startEdit(item)">Editar</button>
              <button class="btn-danger" @click="handleDelete(item.id)">Eliminar</button>
            </td>
          </tr>
        </tbody>
      </table>
    </section>
  </div>
</template>

<script>
export default {
  name: 'App',
  data() {
    return {
      emprendedores: [],
      filterComuna: '',
      filterRubro: '',
      formData: {
        nombre: '',
        comuna: '',
        rubro: '',
        descripcion: '',
        contacto: ''
      },
      editingId: null,
      errorMessage: ''
    };
  },
  mounted() {
    this.fetchEmprendedores();
  },
  methods: {
    async fetchEmprendedores() {
      try {
        const res = await fetch('http://localhost:3000/emprendedores');
        if (res.ok) {
          this.emprendedores = await res.json();
        }
      } catch (err) {
        console.error('Error al conectar con NestJS:', err);
      }
    },

    async handleSearch() {
      try {
        const params = new URLSearchParams();
        if (this.filterComuna.trim()) params.append('comuna', this.filterComuna.trim());
        if (this.filterRubro) params.append('rubro', this.filterRubro);

        const url = params.toString()
          ? `http://localhost:3000/emprendedores/buscar?${params.toString()}`
          : 'http://localhost:3000/emprendedores';

        const res = await fetch(url);
        if (res.ok) {
          this.emprendedores = await res.json();
        }
      } catch (err) {
        console.error('Error al realizar búsqueda:', err);
      }
    },

    resetSearch() {
      this.filterComuna = '';
      this.filterRubro = '';
      this.fetchEmprendedores();
    },

    async handleSubmit() {
      this.errorMessage = '';
      const isEditing = Boolean(this.editingId);
      const url = isEditing
        ? `http://localhost:3000/emprendedores/${this.editingId}`
        : 'http://localhost:3000/emprendedores';

      const method = isEditing ? 'PUT' : 'POST';

      try {
        const res = await fetch(url, {
          method,
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(this.formData)
        });

        const data = await res.json();

        if (!res.ok) {
          const msg = Array.isArray(data.message) ? data.message.join(', ') : (data.message || data.error);
          this.errorMessage = msg || 'Ocurrió un error en la solicitud';
          return;
        }

        this.cancelEdit();
        this.fetchEmprendedores();
      } catch (err) {
        this.errorMessage = 'No fue posible conectar con el servidor';
      }
    },

    startEdit(item) {
      this.editingId = item.id;
      this.formData = {
        nombre: item.nombre,
        comuna: item.comuna,
        rubro: item.rubro,
        descripcion: item.descripcion,
        contacto: item.contacto
      };
      this.errorMessage = '';
      window.scrollTo({ top: 0, behavior: 'smooth' });
    },

    cancelEdit() {
      this.editingId = null;
      this.formData = { nombre: '', comuna: '', rubro: '', descripcion: '', contacto: '' };
      this.errorMessage = '';
    },

    async handleDelete(id) {
      if (!confirm('¿Seguro que deseas eliminar este registro?')) return;

      try {
        const res = await fetch(`http://localhost:3000/emprendedores/${id}`, {
          method: 'DELETE'
        });

        if (!res.ok) {
          alert('Error al eliminar registro');
          return;
        }

        this.fetchEmprendedores();
      } catch (err) {
        console.error('Error al eliminar:', err);
      }
    }
  }
};
</script>

<style scoped>
.app-container {
  max-width: 1000px;
  margin: 0 auto;
  padding: 1.5rem;
  font-family: Arial, sans-serif;
  color: #2c3e50;
}
.app-header {
  border-bottom: 2px solid #eceff1;
  padding-bottom: 0.8rem;
  margin-bottom: 1.5rem;
}
.subtitle {
  color: #607d8b;
  margin: 0.2rem 0 0 0;
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
  grid-template-columns: 1fr 1fr 1fr 1fr;
  gap: 0.8rem;
  margin-bottom: 0.8rem;
}
.filter-grid {
  display: grid;
  grid-template-columns: 2fr 2fr 1fr 1fr;
  gap: 0.8rem;
}
.full-width {
  margin-bottom: 0.8rem;
}
input, select, textarea {
  width: 100%;
  padding: 0.6rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-sizing: border-box;
  font-size: 0.9rem;
}
textarea {
  resize: vertical;
}
.error-banner {
  background-color: #ffebee;
  color: #c62828;
  padding: 0.6rem 1rem;
  border-radius: 4px;
  margin-bottom: 0.8rem;
  font-size: 0.9rem;
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
.btn-search {
  background-color: #1976d2;
  color: white;
  border: none;
  padding: 0.6rem 1.2rem;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}
.badge {
  background-color: #e8f5e9;
  color: #2e7d32;
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
  font-weight: bold;
  font-size: 0.85rem;
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
  padding: 0.35rem 0.6rem;
  border-radius: 4px;
  cursor: pointer;
}
.btn-danger {
  background-color: #e53935;
  color: white;
  border: none;
  padding: 0.35rem 0.6rem;
  border-radius: 4px;
  cursor: pointer;
}
.empty-state {
  color: #777;
  font-style: italic;
}
</style>