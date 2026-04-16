<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import axios from 'axios'

const estado = ref('loading')
const tienda = ref(null)
const errorMsg = ref('')
const tiendaSeleccionada = ref('')
const busqueda = ref('')
const notificaciones = ref([])
const tab = ref('cintillos')
const cintillos = ref([])
const rotulos = ref([])
const cargandoTab = ref(false)
let pollingInterval = null
let notiId = 0

const api = axios.create({ baseURL: 'https://tiendas.multimarcas.app/api' })

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('jwt')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

function tiempoRelativo(fecha) {
  const ahora = new Date()
  const diff = ahora - new Date(fecha)
  const seg = Math.floor(diff / 1000)
  const min = Math.floor(seg / 60)
  const hrs = Math.floor(min / 60)
  const dias = Math.floor(hrs / 24)

  if (seg < 60) return 'Hace unos segundos'
  if (min < 60) return `Hace ${min} min`
  if (hrs < 24) return `Hace ${hrs}h`
  if (dias === 1) return 'Ayer'
  if (dias < 7) return `Hace ${dias} dias`
  return new Date(fecha).toLocaleDateString('es-SV', { day: 'numeric', month: 'short', year: 'numeric' })
}

function formatTipo(tipo) {
  if (!tipo) return ''
  return tipo.replace(/_/g, ' ').replace(/\b\w/g, l => l.toUpperCase())
}

const avatarColores = [
  ['#4f46e5', '#e0e7ff'], // indigo
  ['#0891b2', '#cffafe'], // cyan
  ['#059669', '#d1fae5'], // emerald
  ['#d97706', '#fef3c7'], // amber
  ['#dc2626', '#fee2e2'], // red
  ['#7c3aed', '#ede9fe'], // violet
  ['#db2777', '#fce7f3'], // pink
  ['#2563eb', '#dbeafe'], // blue
  ['#16a34a', '#dcfce7'], // green
  ['#ea580c', '#ffedd5'], // orange
]

function getIniciales(nombre) {
  if (!nombre) return '?'
  const partes = nombre.trim().split(/\s+/)
  if (partes.length >= 2) return (partes[0][0] + partes[partes.length - 1][0]).toUpperCase()
  return partes[0][0].toUpperCase()
}

function getColor(nombre) {
  if (!nombre) return avatarColores[0]
  let hash = 0
  for (let i = 0; i < nombre.length; i++) hash = nombre.charCodeAt(i) + ((hash << 5) - hash)
  return avatarColores[Math.abs(hash) % avatarColores.length]
}

const dataActual = computed(() => tab.value === 'cintillos' ? cintillos.value : rotulos.value)

const dataFiltrada = computed(() => {
  if (!busqueda.value) return dataActual.value
  const q = busqueda.value.toLowerCase()
  return dataActual.value.filter(row =>
    String(row.code).includes(q) ||
    (row.nombre_usuario || '').toLowerCase().includes(q) ||
    (row.comentario || '').toLowerCase().includes(q) ||
    (row.tipo || '').toLowerCase().includes(q)
  )
})

function mostrarNotificacion(mensaje, tipo = 'info') {
  const id = ++notiId
  notificaciones.value.push({ id, mensaje, tipo })
  setTimeout(() => {
    notificaciones.value = notificaciones.value.filter(n => n.id !== id)
  }, 4000)
}

async function autenticar(token) {
  try {
    const { data } = await api.post('/auth', { token })
    localStorage.setItem('jwt', data.token)
    tienda.value = data.tienda
    await cargarTodo()
    estado.value = 'ready'
    iniciarPolling()
  } catch {
    localStorage.removeItem('jwt')
    errorMsg.value = 'Token invalido o tienda inactiva'
    estado.value = 'error'
  }
}

async function cargarTodo() {
  const [c, r] = await Promise.all([
    api.post('/kpi'),
    api.post('/rotulos')
  ])
  cintillos.value = c.data
  rotulos.value = r.data
}

async function cambiarTab(t) {
  if (tab.value === t) return
  cargandoTab.value = true
  tab.value = t
  busqueda.value = ''
  setTimeout(() => { cargandoTab.value = false }, 300)
}

async function verificarNuevos() {
  try {
    const prevCintillos = cintillos.value.length
    const prevRotulos = rotulos.value.length
    await cargarTodo()
    const diffC = cintillos.value.length - prevCintillos
    const diffR = rotulos.value.length - prevRotulos
    if (diffC > 0) mostrarNotificacion(`${diffC} nuevo${diffC > 1 ? 's' : ''} cintillo${diffC > 1 ? 's' : ''}`, 'success')
    if (diffR > 0) mostrarNotificacion(`${diffR} nuevo${diffR > 1 ? 's' : ''} rotulo${diffR > 1 ? 's' : ''}`, 'info')
  } catch {}
}

function iniciarPolling() {
  if (pollingInterval) clearInterval(pollingInterval)
  pollingInterval = setInterval(verificarNuevos, 30000)
}

function seleccionarTienda() {
  if (tiendaSeleccionada.value) autenticar(tiendaSeleccionada.value)
}

function cerrarSesion() {
  localStorage.removeItem('jwt')
  if (pollingInterval) clearInterval(pollingInterval)
  tienda.value = null
  cintillos.value = []
  rotulos.value = []
  estado.value = 'login'
}

onMounted(async () => {
  const params = new URLSearchParams(window.location.search)
  const token = params.get('token')

  if (token) {
    await autenticar(token)
    window.history.replaceState({}, '', window.location.pathname)
    return
  }

  const jwt = localStorage.getItem('jwt')
  if (jwt) {
    try {
      const payload = JSON.parse(atob(jwt.split('.')[1]))
      tienda.value = { id: payload.id, nombre: payload.nombre, codigo: payload.codigo }
      await cargarTodo()
      estado.value = 'ready'
      iniciarPolling()
    } catch {
      localStorage.removeItem('jwt')
      estado.value = 'login'
    }
    return
  }

  estado.value = 'login'
})

onUnmounted(() => {
  if (pollingInterval) clearInterval(pollingInterval)
})
</script>

<template>
  <!-- Notificaciones -->
  <div class="notificaciones">
    <transition-group name="noti">
      <div v-for="noti in notificaciones" :key="noti.id" class="noti" :class="noti.tipo">
        {{ noti.mensaje }}
      </div>
    </transition-group>
  </div>

  <!-- Skeleton global -->
  <div v-if="estado === 'loading'">
    <div class="header">
      <div class="container header-inner">
        <div class="skeleton skeleton-title"></div>
        <div class="skeleton skeleton-badge"></div>
      </div>
    </div>
    <div class="container">
      <div class="tabs-skeleton">
        <div class="skeleton" style="width:100px;height:36px;border-radius:8px"></div>
        <div class="skeleton" style="width:100px;height:36px;border-radius:8px"></div>
      </div>
      <div class="skeleton skeleton-search"></div>
      <div v-for="n in 5" :key="n" class="doc-card">
        <div class="doc-left">
          <div class="skeleton skeleton-avatar"></div>
          <div style="flex:1">
            <div class="skeleton" style="width:140px;height:14px;margin-bottom:8px"></div>
            <div class="skeleton" style="width:100px;height:12px;margin-bottom:6px"></div>
            <div class="skeleton" style="width:80px;height:10px"></div>
          </div>
        </div>
        <div class="skeleton" style="width:100px;height:36px;border-radius:8px"></div>
      </div>
    </div>
  </div>

  <!-- Login -->
  <div v-else-if="estado === 'login'" class="login-screen">
    <div class="login-box">
      <div class="login-icon">
        <svg width="32" height="32" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="1.5">
          <path stroke-linecap="round" stroke-linejoin="round" d="M16.5 10.5V6.75a4.5 4.5 0 10-9 0v3.75m-.75 11.25h10.5a2.25 2.25 0 002.25-2.25v-6.75a2.25 2.25 0 00-2.25-2.25H6.75a2.25 2.25 0 00-2.25 2.25v6.75a2.25 2.25 0 002.25 2.25z" />
        </svg>
      </div>
      <h2>Panel Operador</h2>
      <p>Ingresa tu token de acceso</p>
      <input v-model="tiendaSeleccionada" type="text" placeholder="Token de acceso" @keyup.enter="seleccionarTienda" />
      <button @click="seleccionarTienda" :disabled="!tiendaSeleccionada">Ingresar</button>
      <p v-if="errorMsg" class="login-error">{{ errorMsg }}</p>
    </div>
  </div>

  <!-- Error -->
  <div v-else-if="estado === 'error'" class="error-screen">
    <div class="error-box">
      <h2>Error</h2>
      <p>{{ errorMsg }}</p>
      <button @click="estado = 'login'; errorMsg = ''" class="btn-primary">Volver</button>
    </div>
  </div>

  <!-- Panel -->
  <div v-else-if="estado === 'ready'">
    <div class="header">
      <div class="container header-inner">
        <div class="header-left">
          <h1>{{ tienda.nombre }}</h1>
          <span v-if="tienda.codigo" class="badge-code">{{ tienda.codigo }}</span>
        </div>
        <button @click="cerrarSesion" class="btn-salir">
          <svg width="16" height="16" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
            <path stroke-linecap="round" stroke-linejoin="round" d="M15.75 9V5.25A2.25 2.25 0 0013.5 3h-6a2.25 2.25 0 00-2.25 2.25v13.5A2.25 2.25 0 007.5 21h6a2.25 2.25 0 002.25-2.25V15m3-3h-9m9 0l-3-3m3 3l-3 3" />
          </svg>
          Salir
        </button>
      </div>
    </div>

    <div class="container">
      <!-- Tabs -->
      <div class="tabs">
        <button :class="['tab', { active: tab === 'cintillos' }]" @click="cambiarTab('cintillos')">
          <svg width="16" height="16" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
            <path stroke-linecap="round" stroke-linejoin="round" d="M9.568 3H5.25A2.25 2.25 0 003 5.25v4.318c0 .597.237 1.17.659 1.591l9.581 9.581c.699.699 1.78.872 2.607.33a18.095 18.095 0 005.223-5.223c.542-.827.369-1.908-.33-2.607L11.16 3.66A2.25 2.25 0 009.568 3z" />
            <path stroke-linecap="round" stroke-linejoin="round" d="M6 6h.008v.008H6V6z" />
          </svg>
          Cintillos
          <span class="tab-count">{{ cintillos.length }}</span>
        </button>
        <button :class="['tab', { active: tab === 'rotulos' }]" @click="cambiarTab('rotulos')">
          <svg width="16" height="16" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
            <path stroke-linecap="round" stroke-linejoin="round" d="M3.75 6.75h16.5M3.75 12h16.5m-16.5 5.25H12" />
          </svg>
          Rotulos
          <span class="tab-count">{{ rotulos.length }}</span>
        </button>
      </div>

      <!-- Search -->
      <div class="search-bar">
        <svg class="search-icon" width="18" height="18" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
          <path stroke-linecap="round" stroke-linejoin="round" d="M21 21l-5.197-5.197m0 0A7.5 7.5 0 105.196 5.196a7.5 7.5 0 0010.607 10.607z" />
        </svg>
        <input v-model="busqueda" type="text" :placeholder="`Buscar en ${tab}...`" />
      </div>

      <!-- Info -->
      <div class="results-info">
        {{ dataFiltrada.length }} documento{{ dataFiltrada.length !== 1 ? 's' : '' }}
      </div>

      <!-- Skeleton tab switch -->
      <div v-if="cargandoTab">
        <div v-for="n in 4" :key="n" class="doc-card">
          <div class="doc-left">
            <div class="skeleton skeleton-avatar"></div>
            <div style="flex:1">
              <div class="skeleton" style="width:140px;height:14px;margin-bottom:8px"></div>
              <div class="skeleton" style="width:100px;height:12px;margin-bottom:6px"></div>
              <div class="skeleton" style="width:80px;height:10px"></div>
            </div>
          </div>
          <div class="skeleton" style="width:100px;height:36px;border-radius:8px"></div>
        </div>
      </div>

      <!-- Contenido -->
      <div v-else>
        <!-- Empty -->
        <div v-if="dataFiltrada.length === 0" class="empty-state">
          <svg width="48" height="48" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="1">
            <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 14.25v-2.625a3.375 3.375 0 00-3.375-3.375h-1.5A1.125 1.125 0 0113.5 7.125v-1.5a3.375 3.375 0 00-3.375-3.375H8.25m6.75 12H9.75m3 0H9.75m0 0v3m0-3v-3m10.5-1.5V18a2.25 2.25 0 01-2.25 2.25H5.25A2.25 2.25 0 013 18V6a2.25 2.25 0 012.25-2.25h6.879a3 3 0 012.122.879l5.878 5.878A3 3 0 0121 12.879V18" />
          </svg>
          <p>{{ busqueda ? 'No se encontraron resultados' : 'No hay documentos disponibles' }}</p>
        </div>

        <!-- Cards -->
        <transition-group name="card" tag="div">
          <div v-for="doc in dataFiltrada" :key="doc.id" class="doc-card">
            <div class="doc-left">
              <img v-if="doc.photo" :src="doc.photo" :alt="doc.nombre_usuario" class="avatar" />
              <div v-else class="avatar-placeholder" :style="{ background: getColor(doc.nombre_usuario)[1], color: getColor(doc.nombre_usuario)[0] }">
                {{ getIniciales(doc.nombre_usuario) }}
              </div>
              <div class="doc-info">
                <div class="doc-top">
                  <span class="doc-nombre">{{ doc.nombre_usuario }}</span>
                  <span class="doc-code">#{{ doc.code }}</span>
                </div>
                <p v-if="doc.comentario" class="doc-comentario">{{ doc.comentario }}</p>
                <div v-if="tab === 'rotulos' && doc.tipo" class="doc-tipo">
                  <span class="badge-tipo">{{ formatTipo(doc.tipo) }}</span>
                </div>
                <div class="doc-meta">
                  <span class="doc-fecha">{{ tiempoRelativo(doc.fecha) }}</span>
                  <span v-if="doc.descargado > 0" class="doc-descargas">
                    <svg width="12" height="12" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
                      <path stroke-linecap="round" stroke-linejoin="round" d="M3 16.5v2.25A2.25 2.25 0 005.25 21h13.5A2.25 2.25 0 0021 18.75V16.5M16.5 12L12 16.5m0 0L7.5 12m4.5 4.5V3" />
                    </svg>
                    {{ doc.descargado }}
                  </span>
                </div>
              </div>
            </div>
            <a
              v-if="tab === 'cintillos'"
              :href="`https://docs.multimarcas.app/productos/dl/${doc.slug}`"
              target="_blank"
              class="btn-descargar"
            >
              <svg width="16" height="16" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
                <path stroke-linecap="round" stroke-linejoin="round" d="M3 16.5v2.25A2.25 2.25 0 005.25 21h13.5A2.25 2.25 0 0021 18.75V16.5M16.5 12L12 16.5m0 0L7.5 12m4.5 4.5V3" />
              </svg>
              Descargar
            </a>
            <a
              v-else
              :href="`https://api.multimarcas.app/${doc.path}`"
              target="_blank"
              class="btn-descargar btn-rotulo"
            >
              <svg width="16" height="16" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
                <path stroke-linecap="round" stroke-linejoin="round" d="M3 16.5v2.25A2.25 2.25 0 005.25 21h13.5A2.25 2.25 0 0021 18.75V16.5M16.5 12L12 16.5m0 0L7.5 12m4.5 4.5V3" />
              </svg>
              Descargar
            </a>
          </div>
        </transition-group>
      </div>
    </div>
  </div>
</template>
