<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';
import type { Ref } from 'vue';

// --- Tipado ---
interface Empleado {
  id_empleado: number;
  nombre: string;
}

// --- Configuración ---
const API_URL = 'http://localhost:4001';
const MAX_RETRIES = 3;
const DELAY_MS = 1000;

// --- Estado ---
const empleados: Ref<Empleado[]> = ref([]);
const isLoading = ref(false);
const esError = ref(false);
const mensajeEstado = ref('');
const nombreInput = ref('');
let mensajeTimeoutId: ReturnType<typeof setTimeout> | null = null;

// --- Utilidades ---
const mostrarMensaje = (mensaje: string, error = false) => {
  if (mensajeTimeoutId !== null) clearTimeout(mensajeTimeoutId);
  mensajeEstado.value = mensaje;
  esError.value = error;
  if (!error) {
    mensajeTimeoutId = setTimeout(() => {
      mensajeEstado.value = '';
      mensajeTimeoutId = null;
    }, 4000);
  }
};

const fetchWithRetry = async (url: string, options: any = {}, retries = 0): Promise<any> => {
  try {
    const response = await axios({ url, ...options });
    return response;
  } catch (error) {
    if (retries < MAX_RETRIES) {
      const delay = DELAY_MS * Math.pow(2, retries);
      console.warn(`Intento ${retries + 1} fallido. Reintentando en ${delay}ms...`, error);
      await new Promise(resolve => setTimeout(resolve, delay));
      return fetchWithRetry(url, options, retries + 1);
    } else {
      throw error;
    }
  }
};

// --- Funciones principales ---
const obtenerEmpleados = async () => {
  isLoading.value = true;
  empleados.value = [];
  esError.value = false;
  try {
    const response = await fetchWithRetry(`${API_URL}/empleado`, { method: 'GET' });
    const data: Empleado[] = response.data;
    empleados.value = data.sort((a, b) => a.id_empleado - b.id_empleado);
    mostrarMensaje(`Lista actualizada (${data.length}) empleados`);
  } catch (error: any) {
    console.error(error);
    mostrarMensaje('Error al conectar con el servidor.', true);
    esError.value = true;
  } finally {
    isLoading.value = false;
  }
};

const agregarEmpleado = async () => {
  const nombre = nombreInput.value.trim();
  if (!nombre) {
    mostrarMensaje('El nombre no puede estar vacío.', true);
    return;
  }
  isLoading.value = true;
  try {
    await fetchWithRetry(`${API_URL}/empleado`, { method: 'POST', data: { nombre } });
    nombreInput.value = '';
    mostrarMensaje(`Empleado "${nombre}" agregado ✅`);
    await obtenerEmpleados();
  } catch (error: any) {
    mostrarMensaje('Error al agregar empleado.', true);
  } finally {
    isLoading.value = false;
  }
};

const mensajeClase = computed(() =>
  esError.value
    ? 'bg-red-100 text-red-800 border-red-300'
    : 'bg-emerald-100 text-emerald-800 border-emerald-300'
);

const submitButtonContent = computed(() => {
  if (isLoading.value) {
    return { icon: '🔄', text: 'Guardando...' };
  } else {
    return { icon: '💾', text: 'Guardar Empleado' };
  }
});

onMounted(() => {
  obtenerEmpleados();
});
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-slate-900 via-indigo-950 to-slate-900 text-gray-100 font-inter flex flex-col items-center justify-center py-10 px-6">
    <div class="w-full max-w-4xl bg-white/10 backdrop-blur-md rounded-3xl shadow-2xl border border-white/20 p-10 transition-all hover:shadow-indigo-500/30">
      <h1 class="text-4xl sm:text-5xl font-extrabold text-center bg-clip-text text-transparent bg-gradient-to-r from-emerald-400 via-cyan-400 to-indigo-500 mb-8 drop-shadow-lg">
        Administrador de Personal
      </h1>

      <!-- Formulario -->
      <form @submit.prevent="agregarEmpleado" class="flex flex-col md:flex-row gap-4 mb-8">
        <input
          v-model="nombreInput"
          type="text"
          placeholder="Ej: Sofía Martínez"
          required
          :disabled="isLoading"
          class="flex-1 px-5 py-3 rounded-xl border-0 focus:ring-4 focus:ring-indigo-500/40 text-gray-800 font-medium"
        />
        <button
          type="submit"
          :disabled="isLoading"
          class="flex items-center justify-center gap-2 bg-gradient-to-r from-indigo-600 to-violet-700 text-white font-semibold rounded-xl px-6 py-3 shadow-lg hover:from-indigo-700 hover:to-violet-800 transition-all disabled:opacity-60 disabled:cursor-wait"
        >
          <span>{{ submitButtonContent.icon }}</span>
          <span>{{ submitButtonContent.text }}</span>
        </button>
      </form>

      <!-- Mensaje -->
      <transition name="fade">
        <div
          v-if="mensajeEstado"
          :class="['text-center mb-6 py-3 px-5 rounded-xl border font-semibold shadow-md', mensajeClase]"
        >
          {{ mensajeEstado }}
        </div>
      </transition>

      <!-- Lista -->
      <div class="bg-white/5 rounded-2xl p-6 border border-white/10 shadow-inner">
        <div class="flex justify-between items-center mb-4">
          <h2 class="text-2xl font-bold text-indigo-300 flex items-center gap-2">
            👥 Lista de Empleados
          </h2>
          <button
            @click="obtenerEmpleados"
            :disabled="isLoading"
            class="bg-emerald-500 hover:bg-emerald-600 text-white px-4 py-2 rounded-full font-semibold shadow-md transition disabled:opacity-50"
          >
            🔁 Recargar
          </button>
        </div>

        <!-- Estados -->
        <div v-if="isLoading && empleados.length === 0" class="text-center py-8 text-indigo-400 font-semibold">
          <div class="animate-spin mx-auto h-10 w-10 border-4 border-t-transparent border-indigo-400 rounded-full mb-4"></div>
          Cargando empleados...
        </div>

        <div v-else-if="!isLoading && esError" class="bg-red-100 text-red-700 p-4 rounded-lg text-center">
          ❌ Error al cargar los empleados.
        </div>

        <div v-else-if="!isLoading && empleados.length === 0" class="bg-yellow-100 text-yellow-800 p-4 rounded-lg text-center">
          🟡 No hay empleados registrados todavía.
        </div>

        <!-- Lista de empleados -->
        <ul v-else class="space-y-3">
          <li
            v-for="empleado in empleados"
            :key="empleado.id_empleado"
            class="flex justify-between items-center bg-white/10 backdrop-blur-sm hover:bg-white/20 transition rounded-xl px-5 py-3 border border-white/20"
          >
            <span class="font-medium text-lg">{{ empleado.nombre }}</span>
            <span class="text-sm text-gray-400 font-mono">#{{ empleado.id_empleado }}</span>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.4s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
