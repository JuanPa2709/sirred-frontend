<template>
    <q-page padding>
        <div class="q-mb-md">
            <div class="row justify-between items-center q-mb-md">
                <div class="text-h5 text-weight-medium">Gestión de Cortes (Prácticas)</div>
                <div class="row q-gutter-sm items-center">
                    <q-select
                        v-model="selectedGestion"
                        :options="gestiones"
                        option-label="nombre"
                        label="Gestión Académica"
                        outlined
                        dense
                        bg-color="white"
                        style="min-width: 200px"
                        @update:model-value="loadCortes"
                        :loading="loadingGestiones"
                    >
                        <template v-slot:option="scope">
                            <q-item v-bind="scope.itemProps">
                                <q-item-section>
                                    <q-item-label>{{ scope.opt.nombre }}</q-item-label>
                                    <q-item-label caption>
                                        {{ scope.opt.estado ? 'Activa' : 'Inactiva' }}
                                    </q-item-label>
                                </q-item-section>
                            </q-item>
                        </template>
                    </q-select>
                    <q-btn color="primary" label="Nuevo Corte" icon="add" @click="openDialog()" unelevated :disable="!selectedGestion" />
                </div>
            </div>

            <q-input
                v-model="searchQuery"
                outlined
                dense
                placeholder="Buscar por nombre o fecha..."
                class="q-mb-md"
                clearable
                style="max-width: 400px"
            >
                <template v-slot:prepend>
                    <q-icon name="search" />
                </template>
            </q-input>
        </div>

        <q-table
            :rows="filteredCortes"
            :columns="columns"
            row-key="id"
            :loading="loading"
            flat
            bordered
            :rows-per-page-options="[10, 25, 50]"
            class="shadow-1"
        >
            <template v-slot:body-cell-estado="props">
                <q-td :props="props">
                    <q-badge :color="props.row.estado ? 'positive' : 'grey'">
                        {{ props.row.estado ? 'ACTIVO' : 'Inactivo' }}
                    </q-badge>
                </q-td>
            </template>

            <template v-slot:body-cell-actions="props">
                <q-td :props="props">
                    <q-btn flat dense color="primary" icon="edit" @click="openDialog(props.row)">
                        <q-tooltip>Editar</q-tooltip>
                    </q-btn>
                </q-td>
            </template>
        </q-table>

        <q-dialog v-model="showDialog">
            <q-card style="min-width: 400px">
                <q-card-section class="bg-primary text-white">
                    <div class="text-h6">{{ editMode ? 'Editar Corte' : 'Nuevo Corte' }}</div>
                </q-card-section>

                <q-card-section>
                    <q-form @submit="saveCorte">
                        <q-input v-model="form.nombre" label="Nombre" outlined dense class="q-mb-md"
                            :rules="[val => !!val || 'Requerido']" />
                        <q-input v-model="form.fecha_inicio" label="Fecha de Inicio" type="date" outlined dense
                            class="q-mb-md" :rules="[val => !!val || 'Requerido']" />
                        <q-input v-model="form.fecha_fin" label="Fecha de Fin" type="date" outlined dense
                            class="q-mb-md" />

                        <q-separator class="q-my-sm" />
                        <div class="text-subtitle2 text-primary q-mb-sm">Periodo de Facturación</div>
                        <div class="row q-gutter-sm q-mb-md">
                            <q-input v-model="form.fecha_inicio_facturacion" label="Inicio Facturación" type="date" outlined dense
                                class="col" hint="Desde esta fecha" />
                            <q-input v-model="form.fecha_fin_facturacion" label="Fin Facturación" type="date" outlined dense
                                class="col" hint="Hasta esta fecha" />
                        </div>

                        <q-toggle v-model="form.estado" label="Activar este corte" color="positive" class="q-mb-md" />
                        <q-banner v-if="form.estado" class="bg-info text-white q-mb-md">
                            <template v-slot:avatar>
                                <q-icon name="info" />
                            </template>
                            Para prácticas, se permiten <strong>múltiples cortes activos</strong> simultáneamente. Activar este corte no desactivará los demás.
                        </q-banner>

                        <div class="row q-gutter-sm">
                            <q-btn label="Cancelar" color="grey" flat @click="showDialog = false" />
                            <q-btn type="submit" label="Guardar" color="primary" :loading="saving" />
                        </div>
                    </q-form>
                </q-card-section>
            </q-card>
        </q-dialog>
    </q-page>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { useQuasar } from 'quasar'
import { api } from 'boot/axios'

export default {
    name: 'CortesPracticasPage',
    setup() {
        const $q = useQuasar()
        const cortes = ref([])
        const gestiones = ref([])
        const selectedGestion = ref(null)
        const loadingGestiones = ref(false)
        const loading = ref(false)
        const saving = ref(false)
        const showDialog = ref(false)
        const editMode = ref(false)
        const form = ref({ nombre: '', fecha_inicio: '', fecha_fin: '', fecha_inicio_facturacion: '', fecha_fin_facturacion: '', estado: false })
        const searchQuery = ref('')

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

        const columns = [
            { name: 'nombre', label: 'NOMBRE', field: 'nombre', align: 'left', sortable: true },
            {
                name: 'fecha_inicio',
                label: 'INICIO',
                field: 'fecha_inicio',
                align: 'left',
                sortable: true,
                format: val => formatDate(val)
            },
            {
                name: 'fecha_fin',
                label: 'FIN',
                field: 'fecha_fin',
                align: 'left',
                sortable: true,
                format: val => formatDate(val)
            },
            {
                name: 'periodo_facturacion',
                label: 'FACTURACIÓN',
                align: 'left',
                format: (val, row) => {
                    if (!row.fecha_inicio_facturacion || !row.fecha_fin_facturacion) return 'No definido'
                    return `${formatDate(row.fecha_inicio_facturacion)} - ${formatDate(row.fecha_fin_facturacion)}`
                }
            },
            { name: 'estado', label: 'ESTADO', field: 'estado', align: 'center' },
            { name: 'actions', label: 'ACCIONES', align: 'center' }
        ]

        const filteredCortes = computed(() => {
            if (!searchQuery.value) return cortes.value

            const query = searchQuery.value.toLowerCase()
            return cortes.value.filter(corte =>
                corte.nombre?.toLowerCase().includes(query) ||
                corte.fecha_inicio?.toLowerCase().includes(query) ||
                corte.fecha_fin?.toLowerCase().includes(query)
            )
        })

        const loadGestiones = async () => {
            loadingGestiones.value = true
            try {
                const token = localStorage.getItem('token')
                const response = await api.get('/gestiones', {
                    headers: { Authorization: `Bearer ${token}` }
                })
                gestiones.value = response.data
                if (gestiones.value.length > 0) {
                    const active = gestiones.value.find(g => g.estado)
                    selectedGestion.value = active || gestiones.value[0]
                    loadCortes()
                }
            } catch (error) {
                $q.notify({ type: 'negative', message: 'Error al cargar gestiones' })
            } finally {
                loadingGestiones.value = false
            }
        }

        const loadCortes = async () => {
            if (!selectedGestion.value) {
                cortes.value = []
                return
            }
            loading.value = true
            try {
                const token = localStorage.getItem('token')
                const response = await api.get('/cortes', {
                    params: { tipo_corte: 'PRACTICA', gestion_id: selectedGestion.value.id },
                    headers: { Authorization: `Bearer ${token}` }
                })
                cortes.value = response.data
            } catch (error) {
                $q.notify({ type: 'negative', message: 'Error al cargar cortes' })
            } finally {
                loading.value = false
            }
        }

        const openDialog = (corte = null) => {
            if (corte) {
                form.value = { ...corte, estado: Boolean(corte.estado) }
                editMode.value = true
            } else {
                form.value = { nombre: '', fecha_inicio: '', fecha_fin: '', fecha_inicio_facturacion: '', fecha_fin_facturacion: '', estado: false }
                editMode.value = false
            }
            showDialog.value = true
        }

        const saveCorte = async () => {
            if (!selectedGestion.value) {
                $q.notify({ type: 'warning', message: 'Seleccione una gestión primero' })
                return
            }
            saving.value = true
            try {
                const token = localStorage.getItem('token')
                const payload = { ...form.value, estado: form.value.estado ? 1 : 0, tipo_corte: 'PRACTICA', gestion_id: selectedGestion.value.id }

                if (editMode.value) {
                    await api.put(`/cortes/${form.value.id}`, payload, {
                        headers: { Authorization: `Bearer ${token}` }
                    })
                    $q.notify({ type: 'positive', message: 'Corte actualizado' })
                } else {
                    await api.post('/cortes', payload, {
                        headers: { Authorization: `Bearer ${token}` }
                    })
                    $q.notify({ type: 'positive', message: 'Corte creado' })
                }
                showDialog.value = false
                loadCortes()
            } catch (error) {
                $q.notify({ type: 'negative', message: error.response?.data?.message || 'Error al guardar' })
            } finally {
                saving.value = false
            }
        }

        onMounted(() => {
            loadGestiones()
        })

        return {
            cortes,
            gestiones,
            selectedGestion,
            loadingGestiones,
            loading,
            saving,
            showDialog,
            editMode,
            form,
            columns,
            searchQuery,
            filteredCortes,
            openDialog,
            saveCorte
        }
    }
}
</script>
