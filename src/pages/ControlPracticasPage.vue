<template>
  <q-page padding>
    <div class="q-mb-md">
        <div class="text-h5 text-weight-medium">Control de Facturas (Prácticas Hospitalarias)</div>
        <div class="text-caption text-grey-7">Gestione y apruebe las facturas de prácticas subidas por los docentes</div>
    </div>

    <q-card class="q-mb-md shadow-2" flat bordered>
      <q-card-section class="bg-grey-2">
        <div class="text-subtitle1 text-grey-8">Filtros</div>
      </q-card-section>

      <q-separator />

      <q-card-section>
        <div class="row q-col-gutter-md">
          <div class="col-12 col-sm-6 col-md-2">
            <q-select v-model="selectedCorte" :options="cortes" option-label="nombre" option-value="id" label="Corte"
              outlined dense @update:model-value="loadFacturaciones">
              <template v-slot:prepend>
                <q-icon name="event" />
              </template>
            </q-select>
          </div>

          <div class="col-12 col-sm-6 col-md-2">
            <q-select v-model="selectedSede" :options="sedeOptions" label="Sede" outlined dense clearable
              @update:model-value="onSedeChange">
              <template v-slot:prepend>
                <q-icon name="location_city" />
              </template>
            </q-select>
          </div>

          <div class="col-12 col-sm-6 col-md-2">
            <q-select v-model="selectedCarrera" :options="carreraOptions" label="Carrera" outlined dense clearable>
              <template v-slot:prepend>
                <q-icon name="school" />
              </template>
            </q-select>
          </div>

          <div class="col-12 col-sm-6 col-md-2">
            <q-select v-model="estadoSubida" :options="estadoOptions" label="Estado" outlined dense
              @update:model-value="loadFacturaciones">
              <template v-slot:prepend>
                <q-icon name="filter_list" />
              </template>
            </q-select>
          </div>

          <div class="col-12 col-sm-12 col-md-4">
            <q-input v-model="searchQuery" outlined dense placeholder="Buscar por nombre de docente o CI..." clearable>
              <template v-slot:prepend>
                <q-icon name="search" />
              </template>
            </q-input>
          </div>
        </div>
      </q-card-section>
    </q-card>

    <div class="row justify-between items-center q-mb-md">
      <div class="text-subtitle1 text-grey-8">
        {{ filteredFacturaciones.length }} factura(s) encontrada(s)
      </div>
      <div class="row q-gutter-sm">
        <q-btn v-if="selected.length > 0" color="warning" icon="edit" :label="`Editar (${selected.length})`" unelevated
          @click="openBulkEditDialog">
          <q-tooltip>Editar Sede/Carrera de seleccionados</q-tooltip>
        </q-btn>
        <q-btn color="positive" icon="download" label="Exportar a Excel" unelevated @click="exportToExcel"
          :disable="filteredFacturaciones.length === 0">
          <q-tooltip>Descargar datos filtrados en Excel</q-tooltip>
        </q-btn>
      </div>
    </div>

    <q-table :rows="filteredFacturaciones" :columns="columns" row-key="id" :loading="loading" flat bordered
      :rows-per-page-options="[10, 25, 50, 100]" class="shadow-2" selection="multiple" v-model:selected="selected">
      <template v-slot:body-cell-docente="props">
        <q-td :props="props">
          <div>
            <div class="text-weight-bold">{{ props.row.docente?.apellidos }}</div>
            <div>{{ props.row.docente?.nombre }}</div>
            <div class="text-caption text-grey-7">CI: {{ props.row.docente?.ci }}</div>
          </div>
        </q-td>
      </template>

      <template v-slot:body-cell-sede_carrera="props">
        <q-td :props="props">
          <div>{{ props.row.sede_carrera?.sede?.nombre }}</div>
          <div class="text-caption text-grey-7">{{ props.row.sede_carrera?.carrera?.nombre }}</div>
        </q-td>
      </template>

      <template v-slot:body-cell-materia_hospital="props">
        <q-td :props="props">
          <div class="text-weight-bold">{{ props.row.materia_practica || '-' }}</div>
          <div class="text-caption text-grey-7">{{ props.row.hospital_practica || '-' }}</div>
        </q-td>
      </template>

      <template v-slot:body-cell-fecha_subida="props">
        <q-td :props="props">
          <div v-if="props.row.fecha_subida">
            {{ formatDate(props.row.fecha_subida) }}
          </div>
          <div v-else class="text-grey-6">-</div>
        </q-td>
      </template>

      <template v-slot:body-cell-estado_subida="props">
        <q-td :props="props">
          <q-badge v-if="props.row.estado_subida === null" color="grey-6">
            <q-icon name="schedule" size="xs" class="q-mr-xs" />
            Pendiente
          </q-badge>
          <q-badge v-else-if="props.row.estado_subida === 'SUBIDA'" color="blue">
            <q-icon name="cloud_upload" size="xs" class="q-mr-xs" />
            Subida
          </q-badge>
          <q-badge v-else-if="props.row.estado_subida === 'APROBADO'" color="positive">
            <q-icon name="check_circle" size="xs" class="q-mr-xs" />
            Aprobado
          </q-badge>
          <q-badge v-else-if="props.row.estado_subida === 'DENEGADO'" color="negative">
            <q-icon name="cancel" size="xs" class="q-mr-xs" />
            Denegado
          </q-badge>
          <q-badge v-else-if="props.row.estado_subida === 'RECHAZADO'" color="warning" text-color="black">
            <q-icon name="warning" size="xs" class="q-mr-xs" />
            Rechazado (Auto)
          </q-badge>
          <q-badge v-else-if="props.row.estado_subida === 'REZAGADO'" color="orange">
            <q-icon name="schedule" size="xs" class="q-mr-xs" />
            Rezagado
          </q-badge>
        </q-td>
      </template>

      <template v-slot:body-cell-actions="props">
        <q-td :props="props">
          <div class="row justify-center items-center q-gutter-sm no-wrap">
            <q-btn v-if="props.row.factura_path" flat dense round color="primary" icon="visibility"
              @click="previewFactura(props.row)">
              <q-tooltip>Ver Factura</q-tooltip>
            </q-btn>

            <q-btn v-if="props.row.estado_subida === 'SUBIDA'" flat dense round color="positive" icon="check_circle"
              @click="approveFactura(props.row)">
              <q-tooltip>Aprobar Factura</q-tooltip>
            </q-btn>

            <q-btn v-if="props.row.estado_subida === 'SUBIDA'" flat dense round color="negative" icon="cancel"
              @click="denyFactura(props.row)">
              <q-tooltip>Denegar Factura</q-tooltip>
            </q-btn>
          </div>
        </q-td>
      </template>
    </q-table>

    <!-- PDF Preview Dialog (Drawer Style) -->
    <q-dialog v-model="previewDialog" position="right" full-height>
      <q-card class="column bg-grey-3" style="width: 850px; max-width: 90vw;">
        <q-toolbar class="bg-dark text-white">
          <q-icon name="visibility" size="sm" color="warning" />
          <q-toolbar-title class="text-subtitle1 text-weight-bold text-uppercase tracking-wider">
            Previsualización de Archivo
          </q-toolbar-title>
          
          <q-btn flat round dense icon="open_in_new" @click="downloadCurrentFactura">
             <q-tooltip>Abrir en nueva pestaña</q-tooltip>
          </q-btn>
          <q-btn flat round dense icon="close" v-close-popup>
            <q-tooltip>Cerrar Visor</q-tooltip>
          </q-btn>
        </q-toolbar>

        <q-card-section v-if="currentPreviewFactura" class="bg-grey-2 q-pa-sm border-bottom">
           <div class="row items-center text-caption text-grey-8">
             <q-icon name="person" class="q-mr-xs" size="16px" />
             <span class="text-weight-bold q-mr-sm">Docente:</span> 
             {{ currentPreviewFactura.docente?.nombre }} {{ currentPreviewFactura.docente?.apellidos }}
           </div>
        </q-card-section>

        <q-card-section class="col q-pa-none bg-grey-9 flex flex-center" style="min-height: 400px;">
          <iframe v-if="previewUrl" :src="previewUrl" style="width: 100%; height: 100%; border: none;" title="PDF Preview"></iframe>
        </q-card-section>
      </q-card>
    </q-dialog>

    <!-- Edit Dialog -->
    <q-dialog v-model="editDialog" persistent>
      <q-card style="min-width: 350px">
        <q-card-section>
          <div class="text-h6">{{ isBulkEdit ? 'Editar Múltiples Asignaciones' : 'Editar Asignación' }}</div>
          <div v-if="isBulkEdit" class="text-caption text-grey-7">
            Se actualizarán {{ selected.length }} registros seleccionados.
          </div>
        </q-card-section>

        <q-card-section class="q-pt-none">
          <div class="q-gutter-md">
            <q-select v-model="editForm.sede" :options="allSedes" option-label="nombre" option-value="id" label="Sede"
              outlined dense @update:model-value="onEditSedeChange" />

            <q-select v-model="editForm.carrera" :options="availableCarreras" option-label="nombre" option-value="id"
              label="Carrera" outlined dense :disable="!editForm.sede" />
          </div>
        </q-card-section>

        <q-card-actions align="right" class="text-primary">
          <q-btn flat label="Cancelar" v-close-popup />
          <q-btn flat label="Guardar" @click="saveEdit" :loading="savingEdit" />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { useQuasar } from 'quasar'
import { api } from 'boot/axios'

export default {
  name: 'ControlPracticasPage',
  setup() {
    const $q = useQuasar()
    const facturaciones = ref([])
    const cortes = ref([])
    const selectedCorte = ref(null)
    const selectedSede = ref(null)
    const selectedCarrera = ref(null)
    const estadoSubida = ref({ label: 'Todos', value: null })
    const loading = ref(false)
    const searchQuery = ref('')
    const selected = ref([])

    // PDF Preview State
    const previewDialog = ref(false)
    const previewUrl = ref(null)
    const currentPreviewFactura = ref(null)

    // Edit Dialog State
    const editDialog = ref(false)
    const isBulkEdit = ref(false)
    const savingEdit = ref(false)
    const editForm = ref({
      id: null,
      sede: null,
      carrera: null
    })
    const allSedes = ref([])
    const allCarreras = ref([])
    const availableCarreras = ref([])

    const estadoOptions = [
      { label: 'Todos', value: null },
      { label: 'Pendientes', value: 'null' },
      { label: 'Subidas', value: 'SUBIDA' },
      { label: 'Aprobadas', value: 'APROBADO' },
      { label: 'Denegadas', value: 'DENEGADO' },
      { label: 'Rechazadas (Auto)', value: 'RECHAZADO' },
      { label: 'Rezagadas', value: 'REZAGADO' }
    ]

    const columns = [
      { name: 'docente', label: 'Docente', align: 'left', sortable: true },
      { name: 'sede_carrera', label: 'Sede - Carrera', align: 'left' },
      { name: 'materia_hospital', label: 'Materia / Hospital', align: 'left' },
      { name: 'monto', label: 'Monto', field: 'monto', align: 'right', format: val => `Bs. ${val}`, sortable: true },
      { name: 'fecha_subida', label: 'Fecha de Subida', align: 'center', sortable: true },
      { name: 'estado_subida', label: 'Estado', align: 'center' },
      { name: 'actions', label: 'Acciones', align: 'center' }
    ]

    const sedeOptions = computed(() => {
      const sedes = [...new Set(facturaciones.value.map(f => f.sede_carrera?.sede?.nombre).filter(Boolean))]
      return [{ label: 'Todas', value: null }, ...sedes.map(s => ({ label: s, value: s }))]
    })

    const carreraOptions = computed(() => {
      let filtered = facturaciones.value

      // Filter by selected sede first
      if (selectedSede.value?.value) {
        filtered = filtered.filter(f => f.sede_carrera?.sede?.nombre === selectedSede.value.value)
      }

      const carreras = [...new Set(filtered.map(f => f.sede_carrera?.carrera?.nombre).filter(Boolean))]
      return [{ label: 'Todas', value: null }, ...carreras.map(c => ({ label: c, value: c }))]
    })

    const filteredFacturaciones = computed(() => {
      let filtered = facturaciones.value

      // Filter by sede
      if (selectedSede.value?.value) {
        filtered = filtered.filter(f => f.sede_carrera?.sede?.nombre === selectedSede.value.value)
      }

      // Filter by carrera
      if (selectedCarrera.value?.value) {
        filtered = filtered.filter(f => f.sede_carrera?.carrera?.nombre === selectedCarrera.value.value)
      }

      // Filter by search query
      if (searchQuery.value) {
        const query = searchQuery.value.toLowerCase()
        filtered = filtered.filter(fact => {
          const docenteName = `${fact.docente?.apellidos || ''} ${fact.docente?.nombre || ''}`.toLowerCase()
          const docenteCI = fact.docente?.ci?.toLowerCase() || ''
          return docenteName.includes(query) || docenteCI.includes(query)
        })
      }

      return filtered
    })

    const onSedeChange = () => {
      // Clear carrera selection when sede changes
      selectedCarrera.value = null
    }

    const formatDate = (dateString) => {
      if (!dateString) return '-'
      const date = new Date(dateString)
      const day = String(date.getDate()).padStart(2, '0')
      const month = String(date.getMonth() + 1).padStart(2, '0')
      const year = date.getFullYear()
      const hours = String(date.getHours()).padStart(2, '0')
      const minutes = String(date.getMinutes()).padStart(2, '0')
      return `${day}/${month}/${year} ${hours}:${minutes}`
    }

    const loadCortes = async () => {
      try {
        const token = localStorage.getItem('token')
        const response = await api.get('/cortes', {
          params: { tipo_corte: 'PRACTICA' },
          headers: { Authorization: `Bearer ${token}` }
        })
        cortes.value = response.data
        // Auto-select active corte
        selectedCorte.value = cortes.value.find(c => c.estado === 1) || cortes.value[0]
        if (selectedCorte.value) {
          loadFacturaciones()
        }
      } catch (error) {
        $q.notify({ type: 'negative', message: 'Error al cargar cortes' })
      }
    }

    const loadFacturaciones = async () => {
      if (!selectedCorte.value) return

      loading.value = true
      try {
        const token = localStorage.getItem('token')
        const params = {
          corte_id: selectedCorte.value.id,
          es_practica: true
        }

        if (estadoSubida.value?.value !== null) {
          params.estado_subida = estadoSubida.value.value
        }

        const response = await api.get('/facturaciones', {
          params,
          headers: { Authorization: `Bearer ${token}` }
        })
        facturaciones.value = response.data
      } catch (error) {
        $q.notify({ type: 'negative', message: 'Error al cargar facturaciones' })
      } finally {
        loading.value = false
      }
    }

    const previewFactura = (facturacion) => {
      // Use the API URL from env, stripping '/api' to get the base URL for storage
      const apiUrl = import.meta.env.VITE_API_URL || 'http://localhost:8000/api'
      const baseUrl = apiUrl.replace(/\/api\/?$/, '')
      previewUrl.value = `${baseUrl}/storage/${facturacion.factura_path}`
      currentPreviewFactura.value = facturacion
      previewDialog.value = true
    }

    const downloadCurrentFactura = () => {
      if (previewUrl.value) {
        window.open(previewUrl.value, '_blank')
      }
    }

    const approveFactura = async (facturacion) => {
      $q.dialog({
        title: 'Confirmar Aprobación',
        message: `¿Aprobar la factura de ${facturacion.docente?.nombre} ${facturacion.docente?.apellidos}?`,
        cancel: true,
        persistent: true
      }).onOk(async () => {
        try {
          const token = localStorage.getItem('token')
          await api.post(`/facturaciones/${facturacion.id}/approve`, {}, {
            headers: { Authorization: `Bearer ${token}` }
          })
          $q.notify({ type: 'positive', message: 'Factura aprobada correctamente' })
          loadFacturaciones()
        } catch (error) {
          $q.notify({ type: 'negative', message: error.response?.data?.message || 'Error al aprobar factura' })
        }
      })
    }

    const denyFactura = async (facturacion) => {
      $q.dialog({
        title: 'Confirmar Denegación',
        message: `¿Denegar la factura de ${facturacion.docente?.nombre} ${facturacion.docente?.apellidos}?`,
        cancel: true,
        persistent: true
      }).onOk(async () => {
        try {
          const token = localStorage.getItem('token')
          await api.post(`/facturaciones/${facturacion.id}/deny`, {}, {
            headers: { Authorization: `Bearer ${token}` }
          })
          $q.notify({ type: 'positive', message: 'Factura denegada' })
          loadFacturaciones()
        } catch (error) {
          $q.notify({ type: 'negative', message: 'Error al denegar factura' })
        }
      })
    }

    const exportToExcel = async () => {
      try {
        const token = localStorage.getItem('token')
        const params = {
          corte_id: selectedCorte.value.id,
          es_practica: true
        }

        if (estadoSubida.value?.value !== null && estadoSubida.value?.value !== 'null') {
          params.estado_subida = estadoSubida.value.value
        } else if (estadoSubida.value?.value === 'null') {
          params.estado_subida = 'null'
        }

        if (selectedSede.value?.value) {
          params.sede_nombre = selectedSede.value.value
        }

        if (selectedCarrera.value?.value) {
          params.carrera_nombre = selectedCarrera.value.value
        }

        const response = await api.get('/facturaciones/export', {
          params,
          headers: { Authorization: `Bearer ${token}` },
          responseType: 'blob'
        })

        // Create download link
        const url = window.URL.createObjectURL(new Blob([response.data]))
        const link = document.createElement('a')
        link.href = url

        // Extract filename from response headers or use default
        const contentDisposition = response.headers['content-disposition']
        let filename = 'Facturas_Export.xlsx'
        if (contentDisposition) {
          const filenameMatch = contentDisposition.match(/filename="?(.+)"?/)
          if (filenameMatch) filename = filenameMatch[1]
        }

        link.setAttribute('download', filename)
        document.body.appendChild(link)
        link.click()
        link.remove()
        window.URL.revokeObjectURL(url)

        $q.notify({ type: 'positive', message: 'Excel exportado correctamente' })
      } catch (error) {
        $q.notify({ type: 'negative', message: 'Error al exportar Excel' })
      }
    }

    const loadAllSedesAndCarreras = async () => {
      try {
        const token = localStorage.getItem('token')
        const [sedesRes, carrerasRes] = await Promise.all([
          api.get('/sedes', { headers: { Authorization: `Bearer ${token}` } }),
          api.get('/carreras', { headers: { Authorization: `Bearer ${token}` } })
        ])
        allSedes.value = sedesRes.data
        allCarreras.value = carrerasRes.data
      } catch (error) {
        console.error('Error loading metadata', error)
      }
    }

    const openBulkEditDialog = () => {
      isBulkEdit.value = true
      editForm.value = {
        id: null,
        sede: null,
        carrera: null
      }
      availableCarreras.value = []
      editDialog.value = true
    }

    const onEditSedeChange = (sede) => {
      if (!sede) {
        availableCarreras.value = []
        editForm.value.carrera = null
        return
      }
      availableCarreras.value = allCarreras.value
    }

    const saveEdit = async () => {
      if (!editForm.value.sede || !editForm.value.carrera) {
        $q.notify({ type: 'warning', message: 'Seleccione Sede y Carrera' })
        return
      }

      savingEdit.value = true
      try {
        const token = localStorage.getItem('token')

        const ids = selected.value.map(s => s.id)
        await api.post('/facturaciones/bulk-update', {
          ids,
          sede_id: editForm.value.sede.id,
          carrera_id: editForm.value.carrera.id
        }, {
          headers: { Authorization: `Bearer ${token}` }
        })
        $q.notify({ type: 'positive', message: 'Registros actualizados correctamente' })
        selected.value = [] // Clear selection

        editDialog.value = false
        loadFacturaciones()
      } catch (error) {
        $q.notify({ type: 'negative', message: error.response?.data?.message || 'Error al actualizar' })
      } finally {
        savingEdit.value = false
      }
    }

    onMounted(() => {
      loadCortes()
      loadAllSedesAndCarreras()
    })

    return {
      facturaciones,
      cortes,
      selectedCorte,
      selectedSede,
      selectedCarrera,
      estadoSubida,
      loading,
      searchQuery,
      sedeOptions,
      carreraOptions,
      filteredFacturaciones,
      estadoOptions,
      columns,
      onSedeChange,
      formatDate,
      loadFacturaciones,
      previewFactura,
      previewDialog,
      previewUrl,
      currentPreviewFactura,
      downloadCurrentFactura,
      approveFactura,
      denyFactura,
      exportToExcel,
      editDialog,
      isBulkEdit,
      editForm,
      selected,
      allSedes,
      availableCarreras,
      openBulkEditDialog,
      onEditSedeChange,
      saveEdit,
      savingEdit
    }
  }
}
</script>
