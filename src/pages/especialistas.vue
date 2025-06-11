<template>
  <v-container>
    <v-row justify="space-between" align="center" class="mb-4">
      <v-col>
        <h2>Gestión de Especialistas</h2>
      </v-col>
      <v-col cols="auto" class="d-flex align-center">
        <v-btn icon @click="fetchEspecialistas" class="mr-2" :title="'Refrescar'">
          <v-icon>mdi-refresh</v-icon>
        </v-btn>
        <v-btn color="primary" @click="openDialog()">
          <v-icon left>mdi-plus</v-icon>Nuevo Especialista
        </v-btn>
      </v-col>
    </v-row>

    <v-tabs v-model="tab">
      <v-tab>Activos</v-tab>
      <v-tab>Inactivos</v-tab>
    </v-tabs>

    <v-data-table
      :headers="headers"
      :items="tab === 0 ? especialistas : especialistasInactivos"
      :items-per-page="5"
      class="elevation-1"
      item-value="id"
    >
      <template #item.diasHorarios="{ item }">
        <div v-if="Array.isArray(item.diasHorarios) && item.diasHorarios.length">
          <div v-for="(group, dia) in agruparPorDia(item.diasHorarios)" :key="dia" class="mb-1">
            <v-chip color="primary" size="small" class="mr-1">{{ dia }}</v-chip>
            <span>
              <span v-for="(horario, i) in group" :key="i">
                {{ horario.inicio }}-{{ horario.fin }}<span v-if="i < group.length - 1">,&nbsp;</span>
              </span>
            </span>
          </div>
        </div>
        <span v-else>-</span>
      </template>
      <template #item.actions="{ item }">
        <v-icon 
          small 
          class="mr-2" 
          color="primary" 
          v-if="tab === 0"
          @click="openDialog(item)">
          mdi-pencil
        </v-icon>
        <v-icon
          small
          v-if="tab === 0"
          color="red"
          @click="item.id !== undefined && deleteEspecialista(item.id)"
        >mdi-delete</v-icon>
        <v-icon
          small
          v-if="tab === 1"
          color="green"
          @click="item.id !== undefined && restoreEspecialista(item.id)"
        >mdi-restore</v-icon>
        <v-icon
          small
          v-if="tab === 1"
          color="red"
          @click="item.id !== undefined && forceDeleteEspecialista(item.id)"
        >mdi-delete-forever</v-icon>
      </template>
    </v-data-table>

    <v-dialog v-model="dialog" max-width="700">
      <v-card>
        <v-card-title>
          <span class="text-h5">
            {{ form.id ? 'Editar Especialista' : 'Nuevo Especialista' }}
          </span>
        </v-card-title>
        <v-card-text>
          <v-form ref="formRef" v-model="isValid">
            <v-text-field
              v-model="form.nombreCompleto"
              label="Nombre completo"
              :rules="[v => !!v && v.length >= 3 || 'Mínimo 3 caracteres']"
              required
            />
            <v-text-field
              v-model="form.especialidad"
              label="Especialidad"
              :rules="[v => !!v || 'Requerido']"
              required
            />
            <v-text-field
              v-model="form.registroProfesional"
              label="Registro profesional"
              :rules="[v => !!v || 'Requerido', v => validarRegistroUnico(v)]"
              required
              :disabled="!!form.id"
            />
            <v-divider class="my-4"/>
            <div>
              <strong>Días y horarios de atención</strong>
              <v-row v-for="(dh, idx) in form.diasHorarios" :key="idx" align="center" class="mb-2">
                <v-col cols="4">
                  <v-select
                    :items="diasSemana"
                    v-model="dh.dia"
                    label="Día"
                    dense
                    required
                  />
                </v-col>
                <v-col cols="3">
                  <v-text-field
                    v-model="dh.inicio"
                    label="Inicio"
                    type="time"
                    dense
                    required
                  />
                </v-col>
                <v-col cols="3">
                  <v-text-field
                    v-model="dh.fin"
                    label="Fin"
                    type="time"
                    dense
                    required
                  />
                </v-col>
                <v-col cols="2">
                  <v-btn icon color="red" @click="removeHorario(idx)">
                    <v-icon>mdi-close</v-icon>
                  </v-btn>
                </v-col>
              </v-row>
              <v-btn
                color="primary"
                variant="outlined"
                size="small"
                @click="addHorario"
              >
                <v-icon left>mdi-plus</v-icon>Agregar horario
              </v-btn>
              <div v-if="errorHorarios" class="text-error mt-2">{{ errorHorarios }}</div>
            </div>
          </v-form>
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn text @click="closeDialog()">Cancelar</v-btn>
          <v-btn :disabled="!isValid || !!errorHorarios" color="primary" @click="saveEspecialista()">
            Guardar
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script lang="ts" setup>
import { ref, reactive, onMounted, computed, toRaw, watch } from 'vue'

type Horario = { dia: string, inicio: string, fin: string }
type Especialista = {
  id?: number
  nombreCompleto: string
  especialidad: string
  registroProfesional: string
  diasHorarios: Horario[]
  deletedAt?: string|null
}

const API_BASE = 'http://localhost:3333'

const especialistas = ref<Especialista[]>([])
const especialistasInactivos = ref<Especialista[]>([])
const dialog = ref(false)
const isValid = ref(false)
const formRef = ref()
const tab = ref(0)
const errorHorarios = ref('')
const registroProfesionales = ref<string[]>([])

const diasSemana = [
  'Lunes', 'Martes', 'Miércoles', 'Jueves', 'Viernes', 'Sábado', 'Domingo'
]

const form = reactive<Especialista>({
  nombreCompleto: '',
  especialidad: '',
  registroProfesional: '',
  diasHorarios: []
})

const headers = [
  { title: 'ID', value: 'id' },
  { title: 'Nombre', value: 'nombreCompleto', sortable: true },
  { title: 'Especialidad', value: 'especialidad', sortable: true },
  { title: 'Días y horarios', value: 'diasHorarios' },
  { title: 'Acciones', value: 'actions', sortable: false },
]

// Agrupa horarios por día para la vista
function agruparPorDia(horarios: Horario[]) {
  const agrupado: Record<string, Horario[]> = {}
  for (const h of horarios) {
    if (!agrupado[h.dia]) agrupado[h.dia] = []
    agrupado[h.dia].push(h)
  }
  // Ordena los horarios por inicio
  for (const dia in agrupado) {
    agrupado[dia].sort((a, b) => a.inicio.localeCompare(b.inicio))
  }
  return agrupado
}

// CRUD y helpers
async function fetchEspecialistas() {
  const response = await fetch(`${API_BASE}/especialistas`)
  if (!response.ok) throw new Error('Error al obtener especialistas')
  const data = await response.json()
  especialistas.value = data.filter((e: Especialista) => !e.deletedAt)
  especialistasInactivos.value = data.filter((e: Especialista) => e.deletedAt)
  registroProfesionales.value = data.map((e: Especialista) => e.registroProfesional)
}

function openDialog(item?: Especialista) {
  errorHorarios.value = ''
  if (item) Object.assign(form, JSON.parse(JSON.stringify(item)))
  else Object.assign(form, {
    nombreCompleto: '',
    especialidad: '',
    registroProfesional: '',
    diasHorarios: [],
    activo: true,
    id: undefined
  })
  dialog.value = true
}

function closeDialog() {
  dialog.value = false
  formRef.value?.resetValidation()
}

function addHorario() {
  form.diasHorarios.push({ dia: '', inicio: '', fin: '' })
}

function removeHorario(idx: number) {
  form.diasHorarios.splice(idx, 1)
}

// Validación de traslapes de horarios
function validarHorarios(): string {
  const porDia: Record<string, Horario[]> = {}
  for (const h of form.diasHorarios) {
    if (!h.dia || !h.inicio || !h.fin) return 'Completa todos los campos de horarios'
    if (h.inicio >= h.fin) return 'La hora de inicio debe ser menor que la de fin'
    if (!porDia[h.dia]) porDia[h.dia] = []
    porDia[h.dia].push(h)
  }
  for (const dia in porDia) {
    const horarios = porDia[dia].sort((a, b) => a.inicio.localeCompare(b.inicio))
    for (let i = 1; i < horarios.length; i++) {
      if (horarios[i].inicio < horarios[i - 1].fin) {
        return `Traslape en ${dia}: ${horarios[i - 1].inicio}-${horarios[i - 1].fin} y ${horarios[i].inicio}-${horarios[i].fin}`
      }
    }
  }
  return ''
}

// Validación de registro único
function validarRegistroUnico(value: string) {
  if (!value) return true
  if (!form.id && registroProfesionales.value.includes(value)) return 'Ya existe ese registro profesional'
  return true
}

// Validación reactiva de horarios
watch(() => form.diasHorarios, () => {
  errorHorarios.value = validarHorarios()
}, { deep: true })

async function saveEspecialista() {
  if (!formRef.value) return
  errorHorarios.value = validarHorarios()
  if (errorHorarios.value) return

  const payload = { ...form }
  const body = {
    nombreCompleto: payload.nombreCompleto,
    especialidad: payload.especialidad,
    registroProfesional: payload.registroProfesional,
    diasHorarios: payload.diasHorarios
  }
  const url = form.id ? `${API_BASE}/especialistas/${form.id}` : `${API_BASE}/especialistas`
  const method = form.id ? 'PUT' : 'POST'

  const response = await fetch(url, {
    method,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  })
  if (!response.ok) {
    const err = await response.json()
    alert(err.message || 'Error al guardar especialista')
    return
  }
  await fetchEspecialistas()
  closeDialog()
}

async function deleteEspecialista(id: number) {
  if (!confirm('¿Seguro que deseas inactivar este especialista?')) return
  const response = await fetch(`${API_BASE}/especialistas/${id}`, { method: 'DELETE' })
  if (!response.ok) return alert('Error al eliminar especialista')
  await fetchEspecialistas()
}

async function restoreEspecialista(id: number) {
  if (!confirm('¿Restaurar este especialista?')) return
  const response = await fetch(`${API_BASE}/especialistas/${id}/restore`, { method: 'POST' })
  if (!response.ok) return alert('Error al restaurar especialista')
  await fetchEspecialistas()
}

async function forceDeleteEspecialista(id: number) {
  if (!confirm('¿Eliminar definitivamente este especialista?')) return
  const response = await fetch(`${API_BASE}/especialistas/${id}/force`, { method: 'DELETE' })
  if (!response.ok) return alert('Error al eliminar definitivamente')
  await fetchEspecialistas()
}

onMounted(fetchEspecialistas)
</script>

<style scoped>
.text-error {
  color: #d32f2f;
}
</style>
