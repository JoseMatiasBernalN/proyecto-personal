<template>
  <div class="app">
    <div class="container">
      <!-- HEADER -->
      <header class="header">
        <div>
          <h1 class="title">Panel de Cobranza</h1>
          <p class="subtitle">Gestiona tu cartera y el seguimiento de cada deudor</p>
        </div>
        <div class="header-actions">
          <label class="country" title="Código de país que se usa cuando el teléfono no lo incluye">
            <span>WhatsApp</span>
            <select v-model="paisPorDefecto" aria-label="País por defecto para WhatsApp">
              <option v-for="p in PAISES" :key="p.codigo" :value="p.codigo">
                +{{ p.codigo }} {{ p.nombre }}
              </option>
            </select>
          </label>
          <button class="btn btn-ghost" @click="abrirImport">📥 Importar Excel/CSV</button>
          <button class="btn btn-primary" @click="abrirModal">
            <span class="plus">+</span> Nuevo Deudor
          </button>
        </div>
      </header>

      <!-- KPIs -->
      <section class="kpis">
        <article class="kpi">
          <div class="kpi-icon bg-blue">💰</div>
          <div>
            <p class="kpi-label">Total de Cartera</p>
            <p class="kpi-value">{{ formatoMoneda(totalCartera) }}</p>
            <p class="kpi-hint">Pendiente: {{ formatoMoneda(totalPendiente) }}</p>
          </div>
        </article>
        <article class="kpi">
          <div class="kpi-icon bg-amber">📂</div>
          <div>
            <p class="kpi-label">Casos Activos</p>
            <p class="kpi-value">{{ casosActivos }}</p>
            <p class="kpi-hint">Sin liquidar</p>
          </div>
        </article>
        <article class="kpi">
          <div class="kpi-icon bg-green">✅</div>
          <div>
            <p class="kpi-label">Casos Liquidados</p>
            <p class="kpi-value">{{ casosLiquidados }}</p>
            <p class="kpi-hint">de {{ casos.length }} en total</p>
          </div>
        </article>
      </section>

      <!-- BUSCADOR + FILTROS -->
      <section class="toolbar">
        <div class="search">
          <svg class="search-icon" viewBox="0 0 20 20" fill="currentColor" aria-hidden="true">
            <path
              fill-rule="evenodd"
              d="M8 4a4 4 0 100 8 4 4 0 000-8zM2 8a6 6 0 1110.89 3.476l4.817 4.817a1 1 0 01-1.414 1.414l-4.816-4.816A6 6 0 012 8z"
              clip-rule="evenodd"
            />
          </svg>
          <input
            v-model="busqueda"
            type="text"
            class="search-input"
            placeholder="Buscar por nombre o teléfono..."
            aria-label="Buscar deudores"
          />
          <button v-if="busqueda" class="search-clear" @click="busqueda = ''" aria-label="Limpiar búsqueda">
            ✕
          </button>
        </div>

        <div class="chips">
          <button
            v-for="f in filtros"
            :key="f.valor"
            class="chip"
            :class="{ active: filtroEstado === f.valor }"
            @click="filtroEstado = f.valor"
          >
            {{ f.etiqueta }}
            <span class="chip-count">{{ conteoPorEstado(f.valor) }}</span>
          </button>
        </div>
      </section>

      <!-- LISTADO -->
      <main v-if="casosFiltrados.length" class="grid">
        <article v-for="caso in casosFiltrados" :key="caso.id" class="card">
          <div class="card-top">
            <div class="avatar">{{ iniciales(caso.nombre_deudor) }}</div>
            <div class="card-info">
              <h3 class="card-name">{{ caso.nombre_deudor }}</h3>
              <p class="card-phone">📞 {{ caso.telefono }}</p>
            </div>
            <span class="badge" :class="claseEstado(caso.estado)">
              <span class="dot"></span>{{ caso.estado }}
            </span>
          </div>

          <p class="card-amount" :class="{ paid: caso.estado === 'Liquidado' }">
            {{ formatoMoneda(caso.monto) }}
          </p>

          <div class="card-actions">
            <select
              v-model="caso.estado"
              @change="actualizarEstado(caso)"
              class="select"
              aria-label="Cambiar estado"
            >
              <option value="Por contactar">Por contactar</option>
              <option value="Promesa de pago">Promesa de pago</option>
              <option value="Liquidado">Liquidado</option>
            </select>

            <div class="btn-row">
              <button class="btn btn-whatsapp" @click="abrirWhatsApp(caso)">
                💬 Abrir WhatsApp
              </button>
              <button class="btn btn-ghost" @click="copiarMensaje(caso)" title="Copiar mensaje">
                📋 Copiar
              </button>
            </div>

            <button class="notes-toggle" @click="toggleNotas(caso)">
              <span>📝 Notas <span class="notes-count">{{ caso.total_notas || 0 }}</span></span>
              <span class="chev" :class="{ open: panelesNotas[caso.id]?.abierto }">▾</span>
            </button>

            <div v-if="panelesNotas[caso.id]?.abierto" class="notes">
              <textarea
                v-model="panelesNotas[caso.id].texto"
                rows="2"
                placeholder="Ej: Dijo que paga el viernes a las 3pm"
                @keydown.ctrl.enter="agregarNota(caso)"
              ></textarea>
              <button
                class="btn btn-primary btn-sm"
                :disabled="panelesNotas[caso.id].guardando || !panelesNotas[caso.id].texto.trim()"
                @click="agregarNota(caso)"
              >
                {{ panelesNotas[caso.id].guardando ? 'Guardando...' : 'Guardar nota' }}
              </button>

              <p v-if="panelesNotas[caso.id].cargando" class="notes-empty">Cargando notas...</p>
              <ul v-else-if="panelesNotas[caso.id].lista.length" class="notes-list">
                <li v-for="n in panelesNotas[caso.id].lista" :key="n.id">
                  <time>{{ formatoFecha(n.fecha) }}</time>
                  <p>{{ n.texto }}</p>
                </li>
              </ul>
              <p v-else class="notes-empty">Aún no hay notas para este deudor.</p>
            </div>
          </div>
        </article>
      </main>

      <div v-else class="empty">
        <div class="empty-icon">🔍</div>
        <p class="empty-title">
          {{ casos.length ? 'No hay resultados' : 'Aún no hay deudores registrados' }}
        </p>
        <p class="empty-text">
          {{
            casos.length
              ? 'Prueba con otro nombre, teléfono o filtro.'
              : 'Crea el primero con el botón “+ Nuevo Deudor”.'
          }}
        </p>
      </div>
    </div>

    <!-- MODAL NUEVO DEUDOR -->
    <Transition name="modal">
      <div v-if="modalAbierto" class="overlay" @click.self="cerrarModal">
        <div class="modal" role="dialog" aria-modal="true" aria-labelledby="modal-title">
          <header class="modal-header">
            <h2 id="modal-title">Nuevo Deudor</h2>
            <button class="modal-close" @click="cerrarModal" aria-label="Cerrar">✕</button>
          </header>

          <form class="modal-body" @submit.prevent="guardarCaso" novalidate>
            <div class="field">
              <label for="nombre">Nombre completo</label>
              <input
                id="nombre"
                ref="inputNombre"
                v-model.trim="form.nombre_deudor"
                type="text"
                placeholder="Ej. Juan Pérez"
                :class="{ invalid: errores.nombre_deudor }"
              />
              <small v-if="errores.nombre_deudor" class="error">{{ errores.nombre_deudor }}</small>
            </div>

            <div class="field">
              <label for="telefono">Teléfono</label>
              <input
                id="telefono"
                v-model.trim="form.telefono"
                type="tel"
                inputmode="tel"
                placeholder="Ej. 3001234567"
                :class="{ invalid: errores.telefono }"
              />
              <small v-if="errores.telefono" class="error">{{ errores.telefono }}</small>
            </div>

            <div class="field">
              <label for="monto">Monto adeudado</label>
              <div class="input-prefix">
                <span>$</span>
                <input
                  id="monto"
                  v-model="form.monto"
                  type="number"
                  min="0"
                  step="0.01"
                  inputmode="decimal"
                  placeholder="0.00"
                  :class="{ invalid: errores.monto }"
                />
              </div>
              <small v-if="errores.monto" class="error">{{ errores.monto }}</small>
            </div>

            <footer class="modal-footer">
              <button type="button" class="btn btn-ghost" @click="cerrarModal" :disabled="guardando">
                Cancelar
              </button>
              <button type="submit" class="btn btn-primary" :disabled="guardando">
                {{ guardando ? 'Guardando...' : 'Guardar deudor' }}
              </button>
            </footer>
          </form>
        </div>
      </div>
    </Transition>

    <!-- MODAL IMPORTAR EXCEL / CSV -->
    <Transition name="modal">
      <div v-if="importAbierto" class="overlay" @click.self="cerrarImport">
        <div class="modal" role="dialog" aria-modal="true" aria-labelledby="import-title">
          <header class="modal-header">
            <h2 id="import-title">Importar deudores</h2>
            <button class="modal-close" @click="cerrarImport" aria-label="Cerrar">✕</button>
          </header>

          <div class="modal-body">
            <p class="import-help">
              Sube un archivo <strong>.csv</strong> o <strong>.xlsx</strong> con las columnas
              <code>nombre</code>, <code>telefono</code> y <code>monto</code>.
              <a :href="`${API_URL}/casos/plantilla`" download>Descargar plantilla</a>
            </p>

            <label
              class="dropzone"
              :class="{ over: arrastrando }"
              @dragover.prevent="arrastrando = true"
              @dragleave.prevent="arrastrando = false"
              @drop.prevent="onDrop"
            >
              <input
                ref="inputArchivo"
                type="file"
                accept=".csv,.xlsx,.xlsm"
                hidden
                @change="onArchivoChange"
              />
              <span class="dropzone-icon">📄</span>
              <span v-if="archivoSeleccionado" class="dropzone-file">{{ archivoSeleccionado.name }}</span>
              <span v-else>Arrastra tu archivo aquí o <u>haz clic para elegirlo</u></span>
            </label>

            <p v-if="errorImport" class="import-error">{{ errorImport }}</p>

            <div v-if="resultadoImport" class="import-result">
              <div class="stat ok"><strong>{{ resultadoImport.importados }}</strong> importados</div>
              <div class="stat warn"><strong>{{ resultadoImport.duplicados }}</strong> duplicados omitidos</div>
              <div class="stat bad"><strong>{{ resultadoImport.con_error }}</strong> con error</div>
            </div>
            <ul v-if="resultadoImport?.errores?.length" class="import-errors">
              <li v-for="e in resultadoImport.errores" :key="e.fila">Fila {{ e.fila }}: {{ e.motivo }}</li>
            </ul>

            <footer class="modal-footer">
              <button class="btn btn-ghost" @click="cerrarImport" :disabled="importando">
                {{ resultadoImport ? 'Cerrar' : 'Cancelar' }}
              </button>
              <button
                class="btn btn-primary"
                @click="importarArchivo"
                :disabled="!archivoSeleccionado || importando"
              >
                {{ importando ? 'Importando...' : 'Importar' }}
              </button>
            </footer>
          </div>
        </div>
      </div>
    </Transition>

    <!-- TOAST -->
    <Transition name="toast">
      <div v-if="toast.visible" class="toast" :class="toast.tipo">{{ toast.texto }}</div>
    </Transition>
  </div>
</template>

<script setup>
import { ref, reactive, computed, watch, nextTick, onMounted, onBeforeUnmount } from 'vue';
import axios from 'axios';

const API_URL = 'http://localhost:8000';

/* ---------- Estado ---------- */
const casos = ref([]);
const busqueda = ref('');
const filtroEstado = ref('Todos');

const modalAbierto = ref(false);
const guardando = ref(false);
const inputNombre = ref(null);
const form = reactive({ nombre_deudor: '', telefono: '', monto: '' });
const errores = reactive({ nombre_deudor: '', telefono: '', monto: '' });

const toast = reactive({ visible: false, texto: '', tipo: 'success' });
let toastTimer = null;

const filtros = [
  { valor: 'Todos', etiqueta: 'Todos' },
  { valor: 'Por contactar', etiqueta: 'Por contactar' },
  { valor: 'Promesa de pago', etiqueta: 'Promesa de pago' },
  { valor: 'Liquidado', etiqueta: 'Liquidado' },
];

/* ---------- KPIs ---------- */
const totalCartera = computed(() =>
  casos.value.reduce((acc, c) => acc + Number(c.monto || 0), 0)
);
const totalPendiente = computed(() =>
  casos.value
    .filter((c) => c.estado !== 'Liquidado')
    .reduce((acc, c) => acc + Number(c.monto || 0), 0)
);
const casosActivos = computed(() => casos.value.filter((c) => c.estado !== 'Liquidado').length);
const casosLiquidados = computed(() => casos.value.filter((c) => c.estado === 'Liquidado').length);

/* ---------- Búsqueda y filtros ---------- */
const normalizar = (texto) =>
  String(texto ?? '')
    .toLowerCase()
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '');

const casosFiltrados = computed(() => {
  const q = normalizar(busqueda.value.trim());
  const qTel = busqueda.value.replace(/\D/g, '');

  return casos.value.filter((c) => {
    const coincideEstado = filtroEstado.value === 'Todos' || c.estado === filtroEstado.value;
    if (!coincideEstado) return false;
    if (!q) return true;

    const coincideNombre = normalizar(c.nombre_deudor).includes(q);
    const coincideTel =
      qTel.length > 0 && String(c.telefono ?? '').replace(/\D/g, '').includes(qTel);
    return coincideNombre || coincideTel;
  });
});

const conteoPorEstado = (estado) =>
  estado === 'Todos' ? casos.value.length : casos.value.filter((c) => c.estado === estado).length;

/* ---------- Helpers de UI ---------- */
const formatoMoneda = (valor) => `$${Number(valor || 0).toLocaleString('es-CO')}`;

const claseEstado = (estado) => String(estado).replace(/\s+/g, '-').toLowerCase();

const iniciales = (nombre = '') =>
  nombre
    .split(' ')
    .filter(Boolean)
    .slice(0, 2)
    .map((p) => p[0].toUpperCase())
    .join('');

const mostrarToast = (texto, tipo = 'success') => {
  toast.texto = texto;
  toast.tipo = tipo;
  toast.visible = true;
  clearTimeout(toastTimer);
  toastTimer = setTimeout(() => (toast.visible = false), 2600);
};

/* ---------- API (FastAPI) ---------- */
// Cargar casos desde FastAPI
const cargarCasos = async () => {
  try {
    const response = await axios.get(`${API_URL}/casos/`);
    casos.value = response.data;
  } catch (error) {
    console.error('Error al cargar casos:', error);
    mostrarToast('No se pudo conectar con el servidor', 'error');
  }
};

// Actualizar estado en la base de datos
const actualizarEstado = async (caso) => {
  try {
    await axios.put(`${API_URL}/casos/${caso.id}/estado?nuevo_estado=${caso.estado}`);
    mostrarToast('Estado actualizado');
  } catch (error) {
    console.error('Error al actualizar:', error);
    mostrarToast('No se pudo actualizar el estado', 'error');
    cargarCasos(); // revierte al valor real de la base de datos
  }
};

// Generador de plantilla para WhatsApp
const generarMensaje = (caso) =>
  `Hola ${caso.nombre_deudor}, le escribimos de la agencia de cobranza. Le recordamos que tiene un saldo pendiente de $${caso.monto.toLocaleString()}. Por favor comuníquese con nosotros para brindarle opciones de pago.`;

const copiarMensaje = (caso) => {
  navigator.clipboard
    .writeText(generarMensaje(caso))
    .then(() => mostrarToast('¡Mensaje copiado! Listo para pegar en WhatsApp.'))
    .catch(() => mostrarToast('No se pudo copiar el mensaje', 'error'));
};

/* ---------- WhatsApp directo ---------- */
const PAISES = [
  { codigo: '57', nombre: 'Colombia' },
  { codigo: '52', nombre: 'México' },
  { codigo: '51', nombre: 'Perú' },
];
const paisPorDefecto = ref(localStorage.getItem('pais_whatsapp') || '57');
watch(paisPorDefecto, (valor) => localStorage.setItem('pais_whatsapp', valor));

// wa.me exige el número en formato internacional, solo dígitos y sin "+"
const telefonoParaWhatsApp = (telefono) => {
  const crudo = String(telefono ?? '').trim();
  let digitos = crudo.replace(/\D/g, '');
  if (digitos.startsWith('00')) digitos = digitos.slice(2);

  if (crudo.startsWith('+') || crudo.startsWith('00')) return digitos; // ya trae código de país
  if (digitos.length === 9 && digitos.startsWith('9')) return `51${digitos}`; // móvil de Perú
  if (digitos.length <= 10) return `${paisPorDefecto.value}${digitos}`; // número local
  return digitos; // más de 10 dígitos: se asume que ya incluye el código
};

const abrirWhatsApp = (caso) => {
  const numero = telefonoParaWhatsApp(caso.telefono);
  if (numero.length < 10) {
    mostrarToast('El teléfono de este deudor no es válido', 'error');
    return;
  }
  const url = `https://wa.me/${numero}?text=${encodeURIComponent(generarMensaje(caso))}`;
  window.open(url, '_blank', 'noopener');
  // Deja constancia en el historial (borra esta línea si no quieres el registro automático)
  registrarNotaAutomatica(caso, '📲 Se abrió WhatsApp con el mensaje de cobranza.');
};

/* ---------- Historial de notas ---------- */
const panelesNotas = reactive({}); // { [idCaso]: { abierto, cargando, cargada, guardando, texto, lista } }

const formatoFecha = (iso) =>
  new Date(iso).toLocaleString('es-CO', {
    day: '2-digit',
    month: 'short',
    hour: '2-digit',
    minute: '2-digit',
  });

const cargarNotas = async (id) => {
  const panel = panelesNotas[id];
  panel.cargando = true;
  try {
    const { data } = await axios.get(`${API_URL}/casos/${id}/notas`);
    panel.lista = data;
    panel.cargada = true;
  } catch (error) {
    console.error('Error al cargar notas:', error);
    mostrarToast('No se pudieron cargar las notas', 'error');
  } finally {
    panel.cargando = false;
  }
};

const toggleNotas = async (caso) => {
  if (!panelesNotas[caso.id]) {
    panelesNotas[caso.id] = {
      abierto: false,
      cargando: false,
      cargada: false,
      guardando: false,
      texto: '',
      lista: [],
    };
  }
  const panel = panelesNotas[caso.id];
  panel.abierto = !panel.abierto;
  if (panel.abierto && !panel.cargada) await cargarNotas(caso.id);
};

const agregarNota = async (caso) => {
  const panel = panelesNotas[caso.id];
  const texto = panel.texto.trim();
  if (!texto || panel.guardando) return;

  panel.guardando = true;
  try {
    const { data } = await axios.post(`${API_URL}/casos/${caso.id}/notas`, { texto });
    panel.lista.unshift(data);
    caso.total_notas = (caso.total_notas || 0) + 1;
    panel.texto = '';
    mostrarToast('Nota guardada');
  } catch (error) {
    console.error('Error al guardar la nota:', error);
    mostrarToast('No se pudo guardar la nota', 'error');
  } finally {
    panel.guardando = false;
  }
};

const registrarNotaAutomatica = async (caso, texto) => {
  try {
    const { data } = await axios.post(`${API_URL}/casos/${caso.id}/notas`, { texto });
    caso.total_notas = (caso.total_notas || 0) + 1;
    if (panelesNotas[caso.id]?.cargada) panelesNotas[caso.id].lista.unshift(data);
  } catch (error) {
    console.error('Error al registrar la nota automática:', error);
  }
};

/* ---------- Importación masiva (CSV / Excel) ---------- */
const importAbierto = ref(false);
const importando = ref(false);
const arrastrando = ref(false);
const archivoSeleccionado = ref(null);
const resultadoImport = ref(null);
const errorImport = ref('');
const inputArchivo = ref(null);

const abrirImport = () => {
  archivoSeleccionado.value = null;
  resultadoImport.value = null;
  errorImport.value = '';
  importAbierto.value = true;
};

const cerrarImport = () => {
  if (importando.value) return;
  importAbierto.value = false;
};

const seleccionarArchivo = (archivo) => {
  if (!archivo) return;
  if (!/\.(csv|xlsx|xlsm)$/i.test(archivo.name)) {
    errorImport.value = 'Formato no soportado. Usa un archivo .csv o .xlsx';
    return;
  }
  errorImport.value = '';
  resultadoImport.value = null;
  archivoSeleccionado.value = archivo;
};

const onArchivoChange = (e) => seleccionarArchivo(e.target.files[0]);
const onDrop = (e) => {
  arrastrando.value = false;
  seleccionarArchivo(e.dataTransfer.files[0]);
};

const importarArchivo = async () => {
  if (!archivoSeleccionado.value) return;
  importando.value = true;
  errorImport.value = '';

  const datos = new FormData();
  datos.append('archivo', archivoSeleccionado.value);

  try {
    const { data } = await axios.post(`${API_URL}/casos/importar`, datos);
    resultadoImport.value = data;
    archivoSeleccionado.value = null;
    if (inputArchivo.value) inputArchivo.value.value = '';
    await cargarCasos();
    if (data.importados) mostrarToast(`${data.importados} deudores importados`);
  } catch (error) {
    console.error('Error al importar:', error);
    errorImport.value = error.response?.data?.detail || 'No se pudo importar el archivo';
  } finally {
    importando.value = false;
  }
};

/* ---------- Modal / Formulario ---------- */
const limpiarErrores = () => {
  errores.nombre_deudor = '';
  errores.telefono = '';
  errores.monto = '';
};

const abrirModal = async () => {
  form.nombre_deudor = '';
  form.telefono = '';
  form.monto = '';
  limpiarErrores();
  modalAbierto.value = true;
  await nextTick();
  inputNombre.value?.focus();
};

const cerrarModal = () => {
  if (guardando.value) return;
  modalAbierto.value = false;
};

const validarFormulario = () => {
  limpiarErrores();
  let ok = true;

  if (form.nombre_deudor.length < 3) {
    errores.nombre_deudor = 'Ingresa el nombre completo (mínimo 3 caracteres).';
    ok = false;
  }
  if (form.telefono.replace(/\D/g, '').length < 7) {
    errores.telefono = 'Ingresa un teléfono válido (mínimo 7 dígitos).';
    ok = false;
  }
  const monto = Number(form.monto);
  if (form.monto === '' || Number.isNaN(monto) || monto <= 0) {
    errores.monto = 'Ingresa un monto mayor a 0.';
    ok = false;
  }
  return ok;
};

// Guardar nuevo deudor en la base de datos
const guardarCaso = async () => {
  if (!validarFormulario()) return;

  guardando.value = true;
  try {
    const nuevoCaso = {
      nombre_deudor: form.nombre_deudor,
      telefono: form.telefono,
      monto: Number(form.monto),
    };
    await axios.post(`${API_URL}/casos/`, nuevoCaso);
    modalAbierto.value = false;
    await cargarCasos();
    mostrarToast('Deudor agregado correctamente');
  } catch (error) {
    console.error('Error al guardar el caso:', error);
    mostrarToast('No se pudo guardar el deudor', 'error');
  } finally {
    guardando.value = false;
  }
};

/* ---------- Ciclo de vida ---------- */
const onKeydown = (e) => {
  if (e.key !== 'Escape') return;
  if (importAbierto.value) cerrarImport();
  else if (modalAbierto.value) cerrarModal();
};

onMounted(() => {
  cargarCasos();
  window.addEventListener('keydown', onKeydown);
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeydown);
  clearTimeout(toastTimer);
});
</script>

<style>
/* Estilos globales mínimos */
*,
*::before,
*::after {
  box-sizing: border-box;
}
body {
  font-family: 'Inter', system-ui, -apple-system, 'Segoe UI', sans-serif;
  background-color: #f8fafc;
  color: #0f172a;
  margin: 0;
  -webkit-font-smoothing: antialiased;
}
</style>

<style scoped>
.app {
  min-height: 100vh;
  padding: 32px 20px 56px;
}
.container {
  max-width: 1120px;
  margin: 0 auto;
}

/* Header */
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
  margin-bottom: 28px;
  flex-wrap: wrap;
}
.title {
  font-size: 26px;
  font-weight: 700;
  letter-spacing: -0.02em;
  margin: 0;
}
.subtitle {
  margin: 4px 0 0;
  color: #64748b;
  font-size: 14px;
}

/* Botones */
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  border: none;
  border-radius: 10px;
  padding: 10px 18px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s, transform 0.1s, box-shadow 0.15s;
}
.btn:active:not(:disabled) {
  transform: translateY(1px);
}
.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
.btn-primary {
  background: #4f46e5;
  color: #fff;
  box-shadow: 0 1px 2px rgba(79, 70, 229, 0.35), 0 4px 12px rgba(79, 70, 229, 0.2);
}
.btn-primary:hover:not(:disabled) {
  background: #4338ca;
}
.btn-primary .plus {
  font-size: 18px;
  line-height: 1;
}
.btn-ghost {
  background: #fff;
  color: #334155;
  border: 1px solid #e2e8f0;
}
.btn-ghost:hover:not(:disabled) {
  background: #f1f5f9;
}
.btn-whatsapp {
  background: #16a34a;
  color: #fff;
  flex: 1;
  box-shadow: 0 1px 2px rgba(22, 163, 74, 0.35);
}
.btn-whatsapp:hover {
  background: #15803d;
}
.btn-sm {
  padding: 8px 14px;
  font-size: 13px;
  align-self: flex-end;
}

/* KPIs */
.kpis {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: 16px;
  margin-bottom: 24px;
}
.kpi {
  display: flex;
  align-items: center;
  gap: 16px;
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 16px;
  padding: 20px;
  box-shadow: 0 1px 2px rgba(15, 23, 42, 0.04), 0 8px 24px rgba(15, 23, 42, 0.04);
}
.kpi-icon {
  width: 48px;
  height: 48px;
  border-radius: 12px;
  display: grid;
  place-items: center;
  font-size: 22px;
  flex-shrink: 0;
}
.bg-blue {
  background: #eef2ff;
}
.bg-amber {
  background: #fffbeb;
}
.bg-green {
  background: #ecfdf5;
}
.kpi-label {
  margin: 0;
  font-size: 13px;
  font-weight: 500;
  color: #64748b;
}
.kpi-value {
  margin: 2px 0;
  font-size: 26px;
  font-weight: 700;
  letter-spacing: -0.02em;
}
.kpi-hint {
  margin: 0;
  font-size: 12px;
  color: #94a3b8;
}

/* Toolbar */
.toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  flex-wrap: wrap;
  margin-bottom: 20px;
}
.search {
  position: relative;
  flex: 1 1 300px;
  max-width: 420px;
}
.search-icon {
  position: absolute;
  left: 14px;
  top: 50%;
  transform: translateY(-50%);
  width: 18px;
  height: 18px;
  color: #94a3b8;
  pointer-events: none;
}
.search-input {
  width: 100%;
  padding: 11px 38px 11px 42px;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  background: #fff;
  font-size: 14px;
  color: #0f172a;
  outline: none;
  transition: border-color 0.15s, box-shadow 0.15s;
}
.search-input:focus {
  border-color: #818cf8;
  box-shadow: 0 0 0 4px rgba(99, 102, 241, 0.12);
}
.search-clear {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  border: none;
  background: #f1f5f9;
  color: #64748b;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  font-size: 11px;
  cursor: pointer;
}
.chips {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}
.chip {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 8px 14px;
  border-radius: 999px;
  border: 1px solid #e2e8f0;
  background: #fff;
  color: #475569;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
}
.chip:hover {
  background: #f1f5f9;
}
.chip.active {
  background: #0f172a;
  border-color: #0f172a;
  color: #fff;
}
.chip-count {
  background: rgba(100, 116, 139, 0.15);
  border-radius: 999px;
  padding: 1px 8px;
  font-size: 12px;
}
.chip.active .chip-count {
  background: rgba(255, 255, 255, 0.2);
}

/* Cards */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 20px;
  align-items: start;
}
.card {
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 16px;
  padding: 20px;
  box-shadow: 0 1px 2px rgba(15, 23, 42, 0.04), 0 8px 24px rgba(15, 23, 42, 0.04);
  transition: transform 0.15s, box-shadow 0.15s;
}
.card:hover {
  transform: translateY(-2px);
  box-shadow: 0 2px 4px rgba(15, 23, 42, 0.06), 0 14px 32px rgba(15, 23, 42, 0.08);
}
.card-top {
  display: flex;
  align-items: flex-start;
  gap: 12px;
}
.avatar {
  width: 42px;
  height: 42px;
  border-radius: 50%;
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  color: #fff;
  display: grid;
  place-items: center;
  font-weight: 700;
  font-size: 14px;
  flex-shrink: 0;
}
.card-info {
  flex: 1;
  min-width: 0;
}
.card-name {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.card-phone {
  margin: 4px 0 0;
  font-size: 13px;
  color: #64748b;
}
.card-amount {
  margin: 18px 0 0;
  font-size: 28px;
  font-weight: 700;
  letter-spacing: -0.02em;
  color: #dc2626;
}
.card-amount.paid {
  color: #94a3b8;
  text-decoration: line-through;
}
.card-actions {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-top: 18px;
  padding-top: 16px;
  border-top: 1px solid #f1f5f9;
}
.select {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  background: #fff;
  font-size: 14px;
  font-weight: 500;
  color: #334155;
  cursor: pointer;
  outline: none;
}
.select:focus {
  border-color: #818cf8;
  box-shadow: 0 0 0 4px rgba(99, 102, 241, 0.12);
}

/* Badges */
.badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 600;
  white-space: nowrap;
  border: 1px solid transparent;
}
.badge .dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
}
.badge.por-contactar {
  background: #fef2f2;
  color: #b91c1c;
  border-color: #fecaca;
}
.badge.promesa-de-pago {
  background: #fffbeb;
  color: #b45309;
  border-color: #fde68a;
}
.badge.liquidado {
  background: #ecfdf5;
  color: #047857;
  border-color: #a7f3d0;
}

/* Estado vacío */
.empty {
  text-align: center;
  background: #fff;
  border: 1px dashed #cbd5e1;
  border-radius: 16px;
  padding: 56px 20px;
}
.empty-icon {
  font-size: 36px;
}
.empty-title {
  margin: 12px 0 4px;
  font-size: 16px;
  font-weight: 600;
}
.empty-text {
  margin: 0;
  color: #64748b;
  font-size: 14px;
}

/* Modal */
.overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.5);
  backdrop-filter: blur(4px);
  display: grid;
  place-items: center;
  padding: 20px;
  z-index: 50;
}
.modal {
  width: 100%;
  max-width: 460px;
  background: #fff;
  border-radius: 20px;
  box-shadow: 0 25px 60px rgba(15, 23, 42, 0.3);
  overflow: hidden;
}
.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px 24px;
  border-bottom: 1px solid #f1f5f9;
}
.modal-header h2 {
  margin: 0;
  font-size: 18px;
  font-weight: 700;
}
.modal-close {
  border: none;
  background: #f1f5f9;
  color: #64748b;
  width: 30px;
  height: 30px;
  border-radius: 50%;
  cursor: pointer;
}
.modal-close:hover {
  background: #e2e8f0;
}
.modal-body {
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 18px;
}
.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.field label {
  font-size: 13px;
  font-weight: 600;
  color: #334155;
}
.field input {
  width: 100%;
  padding: 11px 14px;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  font-size: 14px;
  color: #0f172a;
  outline: none;
  transition: border-color 0.15s, box-shadow 0.15s;
}
.field input:focus {
  border-color: #818cf8;
  box-shadow: 0 0 0 4px rgba(99, 102, 241, 0.12);
}
.field input.invalid {
  border-color: #f87171;
}
.input-prefix {
  position: relative;
}
.input-prefix span {
  position: absolute;
  left: 14px;
  top: 50%;
  transform: translateY(-50%);
  color: #94a3b8;
  font-weight: 600;
}
.input-prefix input {
  padding-left: 30px;
}
.error {
  color: #dc2626;
  font-size: 12px;
}
.modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 4px;
}

/* Toast */
.toast {
  position: fixed;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%);
  background: #0f172a;
  color: #fff;
  padding: 12px 20px;
  border-radius: 12px;
  font-size: 14px;
  font-weight: 500;
  box-shadow: 0 10px 30px rgba(15, 23, 42, 0.3);
  z-index: 60;
}
.toast.error {
  background: #b91c1c;
}

/* Header: acciones y país */
.header-actions {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
}
.country {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  font-weight: 600;
  color: #64748b;
}
.country select {
  padding: 9px 10px;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  background: #fff;
  font-size: 13px;
  color: #334155;
  cursor: pointer;
  outline: none;
}

/* Fila de botones de la tarjeta */
.btn-row {
  display: flex;
  gap: 8px;
}

/* Notas */
.notes-toggle {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  padding: 9px 12px;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  background: #f8fafc;
  color: #334155;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
}
.notes-toggle:hover {
  background: #f1f5f9;
}
.notes-count {
  display: inline-block;
  margin-left: 4px;
  min-width: 20px;
  padding: 1px 7px;
  border-radius: 999px;
  background: #e0e7ff;
  color: #4338ca;
  font-size: 12px;
  text-align: center;
}
.chev {
  transition: transform 0.15s;
  color: #94a3b8;
}
.chev.open {
  transform: rotate(180deg);
}
.notes {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.notes textarea {
  width: 100%;
  resize: vertical;
  padding: 10px 12px;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  font: inherit;
  font-size: 13px;
  color: #0f172a;
  outline: none;
}
.notes textarea:focus {
  border-color: #818cf8;
  box-shadow: 0 0 0 4px rgba(99, 102, 241, 0.12);
}
.notes-list {
  list-style: none;
  margin: 0;
  padding: 0;
  max-height: 220px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.notes-list li {
  padding: 9px 12px;
  border-left: 3px solid #c7d2fe;
  background: #f8fafc;
  border-radius: 0 8px 8px 0;
}
.notes-list time {
  font-size: 11px;
  font-weight: 600;
  color: #94a3b8;
}
.notes-list p {
  margin: 3px 0 0;
  font-size: 13px;
  color: #334155;
  white-space: pre-wrap;
  word-break: break-word;
}
.notes-empty {
  margin: 0;
  font-size: 13px;
  color: #94a3b8;
  text-align: center;
}

/* Importación */
.import-help {
  margin: 0;
  font-size: 13px;
  color: #64748b;
  line-height: 1.6;
}
.import-help code {
  background: #f1f5f9;
  padding: 1px 6px;
  border-radius: 6px;
  font-size: 12px;
  color: #334155;
}
.import-help a {
  color: #4f46e5;
  font-weight: 600;
  margin-left: 4px;
}
.dropzone {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
  padding: 28px 16px;
  border: 2px dashed #cbd5e1;
  border-radius: 14px;
  background: #f8fafc;
  color: #64748b;
  font-size: 14px;
  text-align: center;
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s;
}
.dropzone:hover,
.dropzone.over {
  border-color: #818cf8;
  background: #eef2ff;
}
.dropzone-icon {
  font-size: 28px;
}
.dropzone-file {
  color: #0f172a;
  font-weight: 600;
  word-break: break-all;
}
.import-error {
  margin: 0;
  padding: 10px 12px;
  border-radius: 10px;
  background: #fef2f2;
  color: #b91c1c;
  font-size: 13px;
}
.import-result {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
}
.stat {
  padding: 10px;
  border-radius: 10px;
  font-size: 12px;
  text-align: center;
}
.stat strong {
  display: block;
  font-size: 20px;
}
.stat.ok {
  background: #ecfdf5;
  color: #047857;
}
.stat.warn {
  background: #fffbeb;
  color: #b45309;
}
.stat.bad {
  background: #fef2f2;
  color: #b91c1c;
}
.import-errors {
  margin: 0;
  padding: 10px 12px 10px 28px;
  max-height: 120px;
  overflow-y: auto;
  border-radius: 10px;
  background: #fef2f2;
  color: #b91c1c;
  font-size: 12px;
}

/* Transiciones */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s;
}
.modal-enter-active .modal,
.modal-leave-active .modal {
  transition: transform 0.2s;
}
.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}
.modal-enter-from .modal,
.modal-leave-to .modal {
  transform: translateY(12px) scale(0.97);
}
.toast-enter-active,
.toast-leave-active {
  transition: opacity 0.25s, transform 0.25s;
}
.toast-enter-from,
.toast-leave-to {
  opacity: 0;
  transform: translate(-50%, 10px);
}

@media (max-width: 560px) {
  .app {
    padding: 20px 14px 40px;
  }
  .search {
    max-width: none;
  }
}
</style>