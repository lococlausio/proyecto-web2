<script setup>
import { ref, onMounted } from 'vue'

const API = 'http://localhost:3000'
const atractivos = ref([])
const cargando = ref(false)
const error = ref('')

// Estados para búsqueda
const filtroNombre = ref('')
const filtroUbicacion = ref('')

// Estados para nuevo atractivo
const nuevoNombre = ref('')
const nuevaDescripcion = ref('')
const nuevaUbicacion = ref('')
const mensajeForm = ref('')

// Cargar todos los atractivos
async function cargar() {
  try {
    cargando.value = true
    error.value = ''
    const r = await fetch(`${API}/atractivos`)
    atractivos.value = await r.json()
  } catch (e) {
    error.value = 'No se pudo cargar la lista.'
  } finally {
    cargando.value = false
  }
}

// Buscar por query params
async function buscar() {
  try {
    cargando.value = true
    error.value = ''
    const params = new URLSearchParams()
    if (filtroNombre.value.trim()) params.append('nombre', filtroNombre.value.trim())
    if (filtroUbicacion.value.trim()) params.append('ubicacion', filtroUbicacion.value.trim())

    const r = await fetch(`${API}/buscar?${params.toString()}`)
    atractivos.value = await r.json()
  } catch (e) {
    error.value = 'Error al realizar la búsqueda.'
  } finally {
    cargando.value = false
  }
}

function limpiarFiltros() {
  filtroNombre.value = ''
  filtroUbicacion.value = ''
  cargar()
}

// Crear nuevo registro desde el formulario
async function agregarAtractivo() {
  mensajeForm.value = ''
  if (nuevoNombre.value.trim().length < 3) {
    mensajeForm.value = 'El nombre debe tener al menos 3 caracteres.'
    return
  }
  if (nuevaDescripcion.value.trim().length < 10) {
    mensajeForm.value = 'La descripción debe tener al menos 10 caracteres.'
    return
  }

  try {
    const res = await fetch(`${API}/atractivos`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        nombre: nuevoNombre.value,
        descripcion: nuevaDescripcion.value,
        ubicacion: nuevaUbicacion.value
      })
    })

    if (!res.ok) {
      const dataError = await res.json()
      mensajeForm.value = dataError.error || 'Error al guardar.'
      return
    }

    // Limpiar campos del formulario
    nuevoNombre.value = ''
    nuevaDescripcion.value = ''
    nuevaUbicacion.value = ''
    mensajeForm.value = '¡Atractivo agregado con éxito y guardado en Excel!'

    // Recargar lista
    cargar()
  } catch (e) {
    mensajeForm.value = 'Error al conectar con el servidor.'
  }
}

// Eliminar atractivo
async function eliminar(id) {
  const ok = confirm('¿Eliminar este atractivo?')
  if (!ok) return
  await fetch(`${API}/atractivos/${id}`, { method: 'DELETE' })
  atractivos.value = atractivos.value.filter(a => a.id !== id)
}

onMounted(cargar)
</script>

<template>
  <div style="max-width: 700px; margin: 20px auto; font-family: sans-serif;">
    <h2>Gestión Turística - Municipalidad de Chillán</h2>

    <!-- Formulario para crear -->
    <section style="background: #f4f4f4; padding: 15px; border-radius: 8px; margin-bottom: 20px;">
      <h3>Agregar Nuevo Atractivo</h3>
      <form @submit.prevent="agregarAtractivo">
        <div style="margin-bottom: 8px;">
          <input v-model="nuevoNombre" placeholder="Nombre (mín. 3 caracteres)" style="width: 100%; padding: 6px;" />
        </div>
        <div style="margin-bottom: 8px;">
          <input v-model="nuevaDescripcion" placeholder="Descripción (mín. 10 caracteres)" style="width: 100%; padding: 6px;" />
        </div>
        <div style="margin-bottom: 8px;">
          <input v-model="nuevaUbicacion" placeholder="Ubicación (ej. Centro, Cordillera)" style="width: 100%; padding: 6px;" />
        </div>
        <button type="submit" style="padding: 6px 12px; cursor: pointer;">Guardar en Excel</button>
      </form>
      <p v-if="mensajeForm" style="color: blue; margin-top: 8px;">{{ mensajeForm }}</p>
    </section>

    <!-- Filtros de búsqueda -->
    <section style="margin-bottom: 20px; display: flex; gap: 8px;">
      <input v-model="filtroNombre" placeholder="Buscar por nombre..." style="padding: 6px; flex: 1;" />
      <input v-model="filtroUbicacion" placeholder="Buscar por ubicación..." style="padding: 6px; flex: 1;" />
      <button @click="buscar" style="padding: 6px 12px; cursor: pointer;">Buscar</button>
      <button @click="limpiarFiltros" style="padding: 6px 12px; cursor: pointer;">Limpiar</button>
    </section>

    <!-- Listado -->
    <section>
      <h3>Listado de Atractivos</h3>
      <p v-if="cargando">Cargando…</p>
      <p v-if="error" style="color: red;">{{ error }}</p>

      <ul v-if="!cargando && atractivos.length">
        <li v-for="a in atractivos" :key="a.id" style="margin-bottom: 10px;">
          <strong>{{ a.nombre }}</strong> — {{ a.descripcion }}
          <em v-if="a.ubicacion"> ({{ a.ubicacion }})</em>
          <button @click="eliminar(a.id)" style="margin-left: 8px; color: red; cursor: pointer;">Eliminar</button>
        </li>
      </ul>
      <p v-else-if="!cargando">Sin datos encontrados.</p>
    </section>
  </div>
</template>