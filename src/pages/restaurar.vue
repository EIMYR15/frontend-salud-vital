<template>
  <v-container>
    <v-row justify="space-between" align="center" class="mb-4">
      <v-col>
        <h2>Especialistas Eliminados</h2>
      </v-col>
      <v-col cols="auto">
        <v-btn text color="primary" @click="fetchTrashed()">
          <v-icon left>mdi-refresh</v-icon>Refrescar
        </v-btn>
      </v-col>
    </v-row>

    <v-data-table
      :headers="headers"
      :items="trashed"
      :items-per-page="5"
      class="elevation-1"
    >
      <template #item.actions="{ item }">
        <v-btn icon small @click="restore(item.id)">
          <v-icon color="green">mdi-backup-restore</v-icon>
        </v-btn>
        <v-btn icon small @click="forceDelete(item.id)">
          <v-icon color="red">mdi-delete-forever</v-icon>
        </v-btn>
      </template>
    </v-data-table>
  </v-container>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'

type Especialista = {
  id: number
  nombre_completo: string
  especialidad: string
  registro_profesional: string
  dias_horarios: any[]
  activo: boolean
}

const API_BASE = 'http://localhost:3333/api'
const trashed = ref<Especialista[]>([])

const headers = [
  { text: 'ID', value: 'id' },
  { text: 'Nombre', value: 'nombre_completo' },
  { text: 'Especialidad', value: 'especialidad' },
  { text: 'Registro', value: 'registro_profesional' },
  { text: 'Acciones', value: 'actions', sortable: false },
]

async function fetchTrashed() {
  const response = await fetch(`${API_BASE}/especialistas/trashed`)
  if (!response.ok) throw new Error('Error al obtener especialistas eliminados')
  trashed.value = await response.json()
}

async function restore(id: number) {
  if (!confirm('¿Deseas restaurar este especialista?')) return
  const res = await fetch(`${API_BASE}/especialistas/${id}/restore`, { method: 'POST' })
  if (!res.ok) return console.error('Error al restaurar')
  await fetchTrashed()
}

async function forceDelete(id: number) {
  if (!confirm('¿Eliminar permanentemente este especialista? Esta acción es irreversible.')) return
  const res = await fetch(`${API_BASE}/especialistas/${id}/force`, { method: 'DELETE' })
  if (!res.ok) return console.error('Error al eliminar definitivamente')
  await fetchTrashed()
}

onMounted(fetchTrashed)
</script>
