<template>
    <q-page padding>
        <div class="q-mb-md">
            <div class="text-h5 text-weight-medium">Subir Datos (Prácticas Hospitalarias)</div>
            <div class="text-caption text-grey-7">Cargue la información de docentes mediante archivo Excel</div>
        </div>

        <!-- Sin cortes activos -->
        <q-banner v-if="cortesActivos.length === 0" class="bg-warning text-white q-mb-md" rounded>
            <template v-slot:avatar>
                <q-icon name="warning" />
            </template>
            <div class="text-weight-bold">No hay cortes de prácticas activos</div>
            <div class="text-caption">Contacte al administrador para activar un corte de prácticas.</div>
        </q-banner>

        <!-- Cortes activos disponibles -->
        <q-banner v-else class="bg-teal-1 text-teal-9 q-mb-md" rounded>
            <template v-slot:avatar>
                <q-icon name="event" color="teal" />
            </template>
            <span class="text-weight-bold">{{ cortesActivos.length }} corte(s) de prácticas activo(s):</span>
            <span class="q-ml-sm" v-for="c in cortesActivos" :key="c.id">
                <q-badge color="teal" class="q-mr-xs">{{ c.nombre }}</q-badge>
            </span>
        </q-banner>

        <q-card class="shadow-2" flat bordered>
            <q-card-section class="bg-grey-2">
                <div class="text-h6 text-grey-8">Información de Carga</div>
            </q-card-section>

            <q-separator />

            <q-card-section class="q-pa-md">
                <q-form @submit="uploadExcel">
                    <div class="row q-col-gutter-md">

                        <!-- Selector de corte de prácticas -->
                        <div class="col-12">
                            <q-select
                                v-model="selectedCorte"
                                :options="cortesActivos"
                                option-label="nombre"
                                option-value="id"
                                label="Seleccionar Corte de Prácticas *"
                                outlined
                                :disable="cortesActivos.length === 0"
                                :rules="[val => !!val || 'Debe seleccionar un corte']"
                            >
                                <template v-slot:prepend>
                                    <q-icon name="event" color="teal" />
                                </template>
                                <template v-slot:option="scope">
                                    <q-item v-bind="scope.itemProps">
                                        <q-item-section avatar>
                                            <q-icon name="event" color="teal" />
                                        </q-item-section>
                                        <q-item-section>
                                            <q-item-label>{{ scope.opt.nombre }}</q-item-label>
                                            <q-item-label caption>
                                                {{ formatDate(scope.opt.fecha_inicio) }}
                                                {{ scope.opt.fecha_fin ? ' — ' + formatDate(scope.opt.fecha_fin) : '' }}
                                            </q-item-label>
                                        </q-item-section>
                                        <q-item-section side>
                                            <q-badge color="teal">ACTIVO</q-badge>
                                        </q-item-section>
                                    </q-item>
                                </template>
                                <template v-slot:no-option>
                                    <q-item>
                                        <q-item-section class="text-grey">No hay cortes de prácticas activos</q-item-section>
                                    </q-item>
                                </template>
                            </q-select>
                        </div>

                        <div class="col-12 col-md-6">
                            <q-select
                                v-model="selectedSede"
                                :options="sedes"
                                option-label="nombre"
                                option-value="id"
                                label="Seleccionar Sede"
                                outlined
                                @update:model-value="loadCarrerasBySede"
                                :rules="[val => !!val || 'Requerido']"
                            >
                                <template v-slot:prepend>
                                    <q-icon name="location_city" />
                                </template>
                            </q-select>
                        </div>

                        <div class="col-12 col-md-6">
                            <q-select
                                v-model="selectedCarrera"
                                :options="filteredCarreras"
                                option-label="nombre"
                                option-value="id"
                                label="Seleccionar Carrera"
                                outlined
                                use-input
                                input-debounce="300"
                                @filter="filterCarreras"
                                :disable="!selectedSede"
                                :rules="[val => !!val || 'Requerido']"
                            >
                                <template v-slot:prepend>
                                    <q-icon name="school" />
                                </template>
                                <template v-slot:no-option>
                                    <q-item>
                                        <q-item-section class="text-grey">
                                            No se encontraron carreras
                                        </q-item-section>
                                    </q-item>
                                </template>
                            </q-select>
                        </div>

                        <div class="col-12 col-md-6">
                            <q-file
                                v-model="file"
                                label="Archivo Excel"
                                outlined
                                accept=".xlsx, .xls"
                                :rules="[val => !!val || 'Requerido']"
                                max-file-size="5242880"
                                @rejected="onFileRejected"
                            >
                                <template v-slot:prepend>
                                    <q-icon name="attach_file" />
                                </template>
                                <template v-slot:hint>
                                    Tamaño máximo: 5MB
                                </template>
                            </q-file>
                        </div>
                    </div>

                    <q-banner class="bg-blue-1 text-blue-9 q-mt-md" rounded>
                        <template v-slot:avatar>
                            <q-icon name="info" color="blue" />
                        </template>
                        El sistema ahora acepta exactamente el formato original que manejas en Excel (incluyendo celdas combinadas de nombres y apellidos). ¡Sube tu Excel tal cual está!
                    </q-banner>

                    <div class="q-mt-md flex q-gutter-md">
                        <q-btn
                            type="submit"
                            label="Subir Datos"
                            color="primary"
                            icon="upload"
                            :loading="uploading"
                            :disable="cortesActivos.length === 0 || !selectedCorte"
                            unelevated
                            size="md"
                        />
                        <q-btn
                            label="Descargar Plantilla"
                            color="secondary"
                            icon="download"
                            unelevated
                            size="md"
                            @click="downloadTemplate"
                        />
                    </div>
                </q-form>
            </q-card-section>
        </q-card>
    </q-page>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { useQuasar } from 'quasar'
import { api } from 'boot/axios'

export default {
    name: 'UploadPracticasPage',
    setup() {
        const $q = useQuasar()
        const sedes = ref([])
        const cortes = ref([])
        const selectedSede = ref(null)
        const selectedCarrera = ref(null)
        const selectedCorte = ref(null)
        const file = ref(null)
        const uploading = ref(false)
        const allCarreras = ref([])
        const filteredCarreras = ref([])

        // Todos los cortes de prácticas activos (estado = 1)
        const cortesActivos = computed(() => cortes.value.filter(c => c.estado === 1))

        const formatDate = (dateString) => {
            if (!dateString) return ''
            let date
            if (dateString.includes('T')) {
                date = new Date(dateString)
            } else {
                date = new Date(dateString + 'T00:00:00')
            }
            if (isNaN(date.getTime())) return ''
            const day = String(date.getDate()).padStart(2, '0')
            const month = String(date.getMonth() + 1).padStart(2, '0')
            const year = date.getFullYear()
            return `${day}/${month}/${year}`
        }

        const loadSedes = async () => {
            try {
                const token = localStorage.getItem('token')
                const response = await api.get('/sedes', {
                    headers: { Authorization: `Bearer ${token}` }
                })
                sedes.value = response.data
            } catch (error) {
                $q.notify({ type: 'negative', message: 'Error al cargar sedes' })
            }
        }

        const loadCortes = async () => {
            try {
                const token = localStorage.getItem('token')
                const response = await api.get('/cortes', {
                    params: { tipo_corte: 'PRACTICA' },
                    headers: { Authorization: `Bearer ${token}` }
                })
                cortes.value = response.data
            } catch (error) {
                $q.notify({ type: 'negative', message: 'Error al cargar cortes' })
            }
        }

        const loadCarrerasBySede = () => {
            selectedCarrera.value = null
            if (selectedSede.value) {
                allCarreras.value = selectedSede.value.sede_carreras?.map(sc => ({
                    id: sc.id,
                    nombre: sc.carrera?.nombre
                })) || []
                filteredCarreras.value = [...allCarreras.value]
            } else {
                allCarreras.value = []
                filteredCarreras.value = []
            }
        }

        const filterCarreras = (val, update) => {
            update(() => {
                if (val === '') {
                    filteredCarreras.value = allCarreras.value
                } else {
                    const needle = val.toLowerCase()
                    filteredCarreras.value = allCarreras.value.filter(
                        c => c.nombre.toLowerCase().indexOf(needle) > -1
                    )
                }
            })
        }

        const onFileRejected = () => {
            $q.notify({
                type: 'negative',
                message: 'Archivo rechazado. Verifique el formato y tamaño (máx 5MB)'
            })
        }

        const uploadExcel = async () => {
            if (!selectedCorte.value) {
                $q.notify({ type: 'warning', message: 'Debe seleccionar un corte de prácticas' })
                return
            }

            uploading.value = true
            try {
                const token = localStorage.getItem('token')
                const formData = new FormData()
                formData.append('file', file.value)
                formData.append('sede_carrera_id', selectedCarrera.value.id)
                formData.append('corte_id', selectedCorte.value.id)

                await api.post('/facturaciones/upload-excel-practicas', formData, {
                    headers: {
                        Authorization: `Bearer ${token}`,
                        'Content-Type': 'multipart/form-data'
                    }
                })

                $q.notify({ type: 'positive', message: 'Datos importados correctamente' })

                selectedSede.value = null
                selectedCarrera.value = null
                selectedCorte.value = null
                file.value = null
            } catch (error) {
                $q.notify({
                    type: 'negative',
                    message: error.response?.data?.message || 'Error al importar datos'
                })
            } finally {
                uploading.value = false
            }
        }

        const downloadTemplate = () => {
            window.open(`${process.env.VITE_API_URL}/public/template-practicas`, '_blank')
        }

        onMounted(() => {
            loadSedes()
            loadCortes()
        })

        return {
            sedes,
            cortes,
            selectedSede,
            selectedCarrera,
            selectedCorte,
            file,
            uploading,
            cortesActivos,
            allCarreras,
            filteredCarreras,
            formatDate,
            loadCarrerasBySede,
            filterCarreras,
            onFileRejected,
            uploadExcel,
            downloadTemplate
        }
    }
}
</script>

