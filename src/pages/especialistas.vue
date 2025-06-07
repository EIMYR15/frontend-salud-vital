<template>
  <v-container>
    <v-row justify="space-between" align="center" class="mb-4">
      <v-col>
        <h2>Gestión de Especialistas</h2>
      </v-col>
      <v-col cols="auto">
        <v-btn color="primary" @click="openDialog()">
          <v-icon left>mdi-plus</v-icon>Nuevo Especialista
        </v-btn>
      </v-col>
    </v-row>

    <v-data-table
      :headers="headers"
      :items="especialistas"
      :items-per-page="5"
      class="elevation-1"
    >
      <template #item.actions="{ item }">
        <v-icon small class="mr-2" @click="openDialog(item)">mdi-pencil</v-icon>
        <v-icon small @click="deleteEspecialista(item.id)">mdi-delete</v-icon>
      </template>
    </v-data-table>

    <v-dialog v-model="dialog" max-width="600">
      <v-card>
        <v-card-title>
          <span class="text-h5">
            {{ form.id ? 'Editar Especialista' : 'Nuevo Especialista' }}
          </span>
        </v-card-title>

        <v-card-text>
          <v-form ref="formRef" v-model="isValid">
            <v-text-field
              v-model="form.nombre_completo"
              label="Nombre completo"
              :rules="[v => !!v || 'Requerido']"
              required
            />
            <v-text-field
              v-model="form.especialidad"
              label="Especialidad"
              :rules="[v => !!v || 'Requerido']"
              required
            />
            <v-text-field
              v-model="form.registro_profesional"
              label="Registro profesional"
              :rules="[v => !!v || 'Requerido']"
              required
            />
            <v-textarea
              v-model="diasText"
              label="Días y horarios (JSON)"
              auto-grow
              hint='Ej: [{"dia":"Lunes","horario":"9-17"}]'
            />
            <v-switch
              v-model="form.activo"
              label="Activo"
            />
          </v-form>
        </v-card-text>

        <v-card-actions>
          <v-spacer />
          <v-btn text @click="closeDialog()">Cancelar</v-btn>
          <v-btn :disabled="!isValid" color="primary" @click="saveEspecialista()">
            Guardar
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, toRaw, computed } from 'vue'
import type { VForm } from 'vuetify'

type Especialista = {
  id?: number
  nombre_completo: string
  especialidad: string
  registro_profesional: string
  dias_horarios: any[]
  activo: boolean
}

const API_BASE = 'http://localhost:3333/api'
const especialistas = ref<Especialista[]>([])
const dialog = ref(false)
const isValid = ref(false)
const formRef = ref<VForm>()
const form = reactive<Especialista>({
  nombre_completo: '',
  especialidad: '',
  registro_profesional: '',
  dias_horarios: [],
  activo: true,
})

const diasText = computed({
  get: () => JSON.stringify(form.dias_horarios, null, 2),
  set: (val: string) => {
    try { form.dias_horarios = JSON.parse(val) } catch {}
  },
})

const headers = [
  { text: 'ID', value: 'id' },
  { text: 'Nombre', value: 'nombre_completo' },
  { text: 'Especialidad', value: 'especialidad' },
  { text: 'Registro', value: 'registro_profesional' },
  { text: 'Activo', value: 'activo' },
  { text: 'Acciones', value: 'actions', sortable: false },
]

async function fetchEspecialistas() {
  const response = await fetch(`${API_BASE}/especialistas`)
  if (!response.ok) throw new Error('Error al obtener especialistas')
  especialistas.value = await response.json()
}

function openDialog(item?: Especialista) {
  if (item) Object.assign(form, toRaw(item))
  else Object.assign(form, { nombre_completo: '', especialidad: '', registro_profesional: '', dias_horarios: [], activo: true })
  dialog.value = true
}

function closeDialog() {
  dialog.value = false
  formRef.value?.resetValidation()
}

async function saveEspecialista() {
  if (!formRef.value) return
  const payload = { ...form }
  const url = form.id ? `${API_BASE}/especialistas/${form.id}` : `${API_BASE}/especialistas`
  const method = form.id ? 'PUT' : 'POST'

  const response = await fetch(url, {
    method,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
  })
  if (!response.ok) return console.error('Error al guardar especialista')

  await fetchEspecialistas()
  closeDialog()
}

async function deleteEspecialista(id: number) {
  if (!confirm('¿Seguro que deseas eliminar este especialista?')) return

  const response = await fetch(`${API_BASE}/especialistas/${id}`, { method: 'DELETE' })
  if (!response.ok) return console.error('Error al eliminar especialista')

  await fetchEspecialistas()
}

onMounted(fetchEspecialistas)
</script>

<style scoped>
/* Estilos opcionales */
</style>
