<template>
  <q-page class="flex flex-center"
    style="background: linear-gradient(135deg, #663399 0%, #009999 100%); min-height: 100vh;">
    <q-card style="width: 800px; max-width: 95vw;">
      <q-card-section class="bg-primary text-white text-center">
        <div class="text-h5">SIRRED - Búsqueda de Información</div>
        <div class="text-subtitle2">Universidad Técnica Privada Cosmos</div>
      </q-card-section>

      <q-card-section>
        <q-form @submit="searchByCI" class="q-mb-md">
          <div class="row q-gutter-md">
            <q-input v-model="ci" label="Ingrese su CI" outlined dense style="flex: 1"
              :rules="[val => !!val || 'CI es requerido']" />
            <q-btn type="submit" label="Buscar" color="primary" icon="search" :loading="searching" />
          </div>
        </q-form>

        <div v-if="noCorteActivo" class="text-center q-pa-md">
          <q-icon name="info" size="64px" color="warning" />
          <div class="text-h6 q-mt-md">No hay ningún corte activo en este momento</div>
          <div class="text-subtitle2">Por favor, contacte con el administrador</div>
        </div>

        <div v-else-if="searchResult">
          <q-card class="bg-primary text-white q-mb-md shadow-2">
            <q-card-section>
              <div class="row items-center no-wrap">
                <q-avatar size="64px" font-size="42px" color="white" text-color="primary" icon="person"
                  class="q-mr-md" />
                <div>
                  <div class="text-h6 text-weight-bold">{{ searchResult.docente.nombre }} {{
                    searchResult.docente.apellidos }}</div>
                  <div class="text-subtitle2 opacity-80">CI: {{ searchResult.docente.ci }}</div>
                  <div v-for="corte in searchResult.cortes_activos" :key="corte.id" class="q-mt-sm">
                    <div class="text-subtitle1">
                      <q-badge :color="corte.tipo_corte === 'PRACTICA' ? 'purple' : 'secondary'" class="q-pa-xs text-body2">
                        {{ corte.nombre }}
                      </q-badge>
                    </div>
                    <div class="text-caption text-italic opacity-80 q-mt-xs">
                      Periodo: {{ formatDateDisplay(corte.fecha_inicio) }} -
                      {{ corte.fecha_fin ? formatDateDisplay(corte.fecha_fin) : 'Indefinido' }}
                    </div>
                  </div>
                </div>
              </div>
            </q-card-section>
          </q-card>

          <!-- ─── BANNER NIT: Sin verificar (null) ─────────────────────────────── -->
          <q-banner v-if="!searchResult.nit_info?.estado_nit" rounded
            class="bg-purple-1 text-purple-9 q-mb-md" style="border: 1.5px solid #7b1fa2">
            <template v-slot:avatar>
              <q-icon name="shield" color="purple" size="32px" />
            </template>
            <div class="text-weight-bold text-subtitle2">Verificación de NIT Requerida</div>
            <div class="q-mt-xs">Para habilitar sus pagos, debe verificar su NIT por única vez. Suba su
              <strong>Documento de Exhibición de NIT</strong> del portal de Impuestos Nacionales (SIAT).</div>
            <template v-slot:action>
              <q-btn flat color="purple" label="¿Cómo obtenerlo?" icon="help_outline" class="q-mr-sm"
                @click="showNitHelpDialog = true" />
              <q-btn flat color="purple" label="Verificar mi NIT ahora" icon="upload_file"
                @click="showNitDialog = true" />
            </template>
          </q-banner>

          <!-- ─── BANNER NIT: Pendiente ─────────────────────────────────────────── -->
          <q-banner v-else-if="searchResult.nit_info?.estado_nit === 'PENDIENTE'" rounded
            class="bg-amber-1 text-amber-9 q-mb-md" style="border: 1.5px solid #f59e0b">
            <template v-slot:avatar>
              <q-icon name="pending" color="amber-8" size="28px" />
            </template>
            <div class="text-weight-bold">Verificación de NIT Pendiente</div>
            <div class="text-caption q-mt-xs">No fue posible conectar con Impuestos Nacionales al momento de subir
              su documento. Su factura puede ser subida, pero el pago quedará en revisión hasta que un
              administrador verifique su NIT manualmente.</div>
          </q-banner>

          <!-- ─── BANNER NIT: Inválido ──────────────────────────────────────────── -->
          <q-banner v-else-if="searchResult.nit_info?.estado_nit === 'INVALIDO'" rounded
            class="bg-red-1 text-red-9 q-mb-md" style="border: 1.5px solid #c62828">
            <template v-slot:avatar>
              <q-icon name="gpp_bad" color="red" size="28px" />
            </template>
            <div class="text-weight-bold">NIT con Actividad de Docencia No Vigente</div>
            <div class="text-caption q-mt-xs">Según el SIAT (Impuestos Nacionales), su NIT no cuenta con la
              actividad económica de <strong>Docencia</strong> vigente. No puede subir facturas hasta regularizar
              su situación en Impuestos Nacionales.</div>
          </q-banner>

          <!-- Banner Periodo de Facturación -->
          <q-banner v-if="searchResult.periodo_facturacion?.fecha_inicio" rounded class="q-mb-md"
            :class="getPeriodoBannerClass()">
            <template v-slot:avatar>
              <q-icon :name="getPeriodoIcon()" />
            </template>
            <div class="text-weight-bold">{{ getPeriodoTitle() }}</div>
            <div>Periodo: {{ formatDate(searchResult.periodo_facturacion.fecha_inicio) }} - {{
              formatDate(searchResult.periodo_facturacion.fecha_fin) }}</div>
            <div
              v-if="searchResult.periodo_facturacion.estado === 'abierto' && searchResult.periodo_facturacion.dias_restantes !== null"
              class="text-caption q-mt-xs">
              <q-icon name="schedule" size="xs" /> {{ searchResult.periodo_facturacion.dias_restantes }} día(s)
              restantes para subir
              su factura
            </div>
          </q-banner>

          <div v-for="facturacion in searchResult.facturaciones" :key="facturacion.id" class="q-mb-md">
            <q-card bordered>
              <q-card-section class="bg-grey-2">
                <div class="row justify-between items-center">
                  <div>
                    <div class="text-subtitle1">
                      <strong>{{ facturacion.sede_carrera.sede.nombre }}</strong> - {{ facturacion.sede_carrera.carrera.nombre }}
                    </div>
                    <div class="text-caption text-weight-bold" v-if="facturacion.es_practica">
                      Prácticas: {{ facturacion.materia_practica }} ({{ facturacion.hospital_practica }})
                    </div>
                    <div class="text-caption q-mt-xs">
                      <q-badge color="dark" class="q-mr-xs">{{ facturacion.corte?.nombre }}</q-badge>
                    </div>
                  </div>
                  <div class="text-right">
                    <div class="text-h6">Bs. {{ facturacion.monto }}</div>
                    <div class="text-caption" v-if="!facturacion.es_practica">Carga: {{ facturacion.carga_horaria }} hrs</div>
                  </div>
                </div>
              </q-card-section>

              <q-card-section v-if="facturacion.tipo_contrato === 'FACTURACION'">
                <!-- NIT INVALIDO: bloqueo dentro de la card de facturacion -->
                <q-banner v-if="searchResult.nit_info?.estado_nit === 'INVALIDO'" rounded
                  class="bg-red-1 text-red-9 q-mb-md">
                  <template v-slot:avatar><q-icon name="block" color="red" /></template>
                  <div class="text-weight-bold">Subida de Factura Bloqueada</div>
                  <div>Regularice su actividad de Docencia en el SIAT antes de continuar.</div>
                </q-banner>

                <!-- NIT SIN VERIFICAR: bloqueo dentro de la card de facturacion -->
                <q-banner v-else-if="!searchResult.nit_info?.estado_nit" rounded
                  class="bg-purple-1 text-purple-9 q-mb-md">
                  <template v-slot:avatar><q-icon name="shield" color="purple" /></template>
                  <div class="text-weight-bold">Verifique su NIT para habilitar esta opción</div>
                  <div>Haga clic en el botón "Verificar mi NIT ahora" que aparece arriba.</div>
                </q-banner>

                <!-- Estado REZAGADO (solo si NIT es válido) -->
                  <q-banner v-if="facturacion.estado_subida === 'REZAGADO'" rounded
                  class="bg-orange-1 text-orange-9 q-mb-md">
                  <template v-slot:avatar>
                    <q-icon name="schedule" color="orange" />
                  </template>
                  <div class="text-weight-bold">Factura Rezagada</div>
                  <div>No subió su factura dentro del periodo establecido. Aún puede subirla ahora.</div>
                </q-banner>

                <!-- Periodo Cerrado (no es rezagado) -->
                <q-banner v-if="searchResult.periodo_facturacion?.estado === 'cerrado' && !facturacion.estado_subida"
                  rounded class="bg-red-1 text-red-9 q-mb-md">
                  <template v-slot:avatar>
                    <q-icon name="block" color="red" />
                  </template>
                  <div class="text-weight-bold">Periodo de Facturación Cerrado</div>
                  <div>El plazo para subir facturas ha terminado. Contacte al administrador.</div>
                </q-banner>

                <div
                  v-if="(!facturacion.estado_subida && searchResult.periodo_facturacion?.estado !== 'cerrado') || facturacion.estado_subida === 'DENEGADO' || facturacion.estado_subida === 'REZAGADO' || facturacion.estado_subida === 'RECHAZADO'">
                  <!-- Banner para DENEGADO (administrador) -->
                  <q-banner v-if="facturacion.estado_subida === 'DENEGADO'" rounded
                    class="bg-red-1 text-red-9 q-mb-md border-red">
                    <template v-slot:avatar>
                      <q-icon name="error" color="red" />
                    </template>
                    <div class="text-weight-bold">Factura Denegada por Administrador</div>
                    <div>Su factura fue denegada. Por favor, revise los requisitos y suba una nueva.
                    </div>
                    <div class="text-caption q-mt-xs" v-if="facturacion.fecha_subida">
                      Fecha de subida anterior: {{ new
                        Date(facturacion.fecha_subida).toLocaleDateString() }}
                    </div>
                  </q-banner>

                  <!-- Banner para RECHAZADO (validación automática) -->
                  <q-banner v-if="facturacion.estado_subida === 'RECHAZADO'" rounded
                    class="bg-orange-1 text-orange-9 q-mb-md">
                    <template v-slot:avatar>
                      <q-icon name="warning" color="orange" />
                    </template>
                    <div class="text-weight-bold">Factura Rechazada - Errores de Validación</div>

                    <!-- Feedback Detallado (persistente) -->
                    <div class="q-mt-sm"
                      v-if="facturacion.errores_validacion?.length && typeof facturacion.errores_validacion[0] === 'object'">
                      <q-list dense bordered separator class="bg-white rounded-borders q-my-sm">
                        <q-item v-for="(detalle, idx) in facturacion.errores_validacion" :key="idx"
                          :class="detalle.estado === 'ok' ? 'bg-green-1' : 'bg-red-1'">
                          <q-item-section avatar style="min-width: 30px; padding-right: 0;">
                            <q-icon :name="detalle.estado === 'ok' ? 'check_circle' : 'cancel'"
                              :color="detalle.estado === 'ok' ? 'positive' : 'negative'" size="xs" />
                          </q-item-section>
                          <q-item-section>
                            <q-item-label :class="detalle.estado === 'ok' ? 'text-positive' : 'text-negative'"
                              class="text-weight-bold text-caption">{{ detalle.titulo }}</q-item-label>
                            <q-item-label caption style="font-size: 11px; line-height: 1.2;">{{ detalle.mensaje
                            }}</q-item-label>
                          </q-item-section>
                        </q-item>
                      </q-list>
                    </div>

                    <!-- Fallback para errores antiguos (strings simples) -->
                    <div class="q-mt-sm" v-else-if="facturacion.errores_validacion?.length">
                      <div v-for="(error, idx) in facturacion.errores_validacion" :key="idx" class="q-mb-xs">
                        • {{ error }}
                      </div>
                    </div>

                    <div class="text-caption q-mt-sm">
                      <q-icon name="info" size="xs" /> Por favor, corrija los errores y vuelva a subir su factura.
                    </div>
                  </q-banner>

                  <div class="row items-center q-col-gutter-md">
                    <div class="col-12 col-sm-8">
                      <q-file v-model="facturacion.file" label="Seleccionar Factura (PDF)" outlined dense accept=".pdf"
                        max-file-size="2097152" @rejected="onRejected">
                        <template v-slot:prepend>
                          <q-icon name="attach_file" />
                        </template>
                      </q-file>
                      <div class="text-caption text-grey-7 q-mt-xs">
                        Formato PDF, máx 2MB
                      </div>
                    </div>
                    <div class="col-12 col-sm-4">
                      <q-btn label="Subir Factura" color="primary" icon="cloud_upload" class="full-width"
                        @click="uploadFactura(facturacion)" :loading="uploading[facturacion.id]"
                        :disable="!facturacion.file" />
                    </div>
                  </div>
                </div>

                <div v-else-if="facturacion.estado_subida === 'SUBIDA'">
                  <q-card flat bordered class="bg-blue-1 border-blue">
                    <q-card-section>
                      <div class="row items-center justify-between q-mb-md">
                        <div class="row items-center">
                          <q-icon name="cloud_upload" color="blue" size="32px" class="q-mr-md" />
                          <div>
                            <div class="text-subtitle1 text-weight-bold text-blue">
                              Factura Subida Correctamente
                            </div>
                            <div class="text-caption text-grey-8">
                              Registrada el {{ new Date(facturacion.fecha_subida).toLocaleDateString() }}
                            </div>
                          </div>
                        </div>
                        <q-btn flat round color="primary" icon="visibility" @click="viewFactura(facturacion)">
                          <q-tooltip>Ver Factura</q-tooltip>
                        </q-btn>
                      </div>

                      <q-banner class="bg-warning text-white q-mb-md" rounded>
                        <template v-slot:avatar>
                          <q-icon name="info" />
                        </template>
                        <div class="text-weight-bold">¿Necesita reemplazar su factura?</div>
                        <div class="text-caption">Si detectó un error, puede subir una nueva factura. La anterior será
                          reemplazada.</div>
                      </q-banner>

                      <div class="row items-center q-col-gutter-md">
                        <div class="col-12 col-sm-8">
                          <q-file v-model="facturacion.file" label="Seleccionar Nueva Factura (PDF)" outlined dense
                            accept=".pdf" max-file-size="2097152" @rejected="onRejected">
                            <template v-slot:prepend>
                              <q-icon name="attach_file" />
                            </template>
                          </q-file>
                          <div class="text-caption text-grey-7 q-mt-xs">
                            Formato PDF, máx 2MB
                          </div>
                        </div>
                        <div class="col-12 col-sm-4">
                          <q-btn label="Reemplazar Factura" color="warning" icon="sync" class="full-width"
                            @click="confirmReplaceFactura(facturacion)" :loading="uploading[facturacion.id]"
                            :disable="!facturacion.file" />
                        </div>
                      </div>
                    </q-card-section>
                  </q-card>
                </div>

                <div v-else-if="facturacion.estado_subida === 'APROBADO'">
                  <q-card flat bordered class="bg-green-1 border-green">
                    <q-card-section class="row items-center justify-between">
                      <div class="row items-center">
                        <q-icon name="check_circle" color="positive" size="32px" class="q-mr-md" />
                        <div>
                          <div class="text-subtitle1 text-weight-bold text-positive">
                            Factura Aprobada
                          </div>
                          <div class="text-caption text-grey-8">
                            Registrada el {{ new Date(facturacion.fecha_subida).toLocaleDateString() }}
                          </div>
                          <div class="text-caption text-positive q-mt-xs">
                            <q-icon name="lock" size="xs" /> No se puede modificar una factura aprobada
                          </div>
                        </div>
                      </div>
                      <q-btn flat round color="primary" icon="visibility" @click="viewFactura(facturacion)">
                        <q-tooltip>Ver Factura</q-tooltip>
                      </q-btn>
                    </q-card-section>
                  </q-card>
                </div>
              </q-card-section>

              <q-card-section v-else>
                <q-banner class="bg-info text-white">
                  <template v-slot:avatar>
                    <q-icon name="info" />
                  </template>
                  Este tipo de contrato es solo informativo. No requiere subir factura.
                </q-banner>
              </q-card-section>
            </q-card>
          </div>
        </div>
      </q-card-section>


    </q-card>

    <!-- Dialogo de Verificación de NIT -->
    <q-dialog v-model="showNitDialog" persistent>
      <q-card style="min-width: 440px; max-width: 95vw; border-radius: 16px;">
        <q-card-section class="bg-purple text-white q-py-md">
          <div class="row items-center no-wrap">
            <q-icon name="shield" size="28px" class="q-mr-sm" />
            <div>
              <div class="text-h6 text-weight-bold">Verificación de NIT</div>
              <div class="text-caption opacity-80">Solo se realiza una vez</div>
            </div>
          </div>
        </q-card-section>

        <q-card-section class="q-pt-md">
          <!-- Step 1: Instructions -->
          <div v-if="nitStep === 1">
            <q-banner rounded class="bg-blue-1 text-blue-9 q-mb-md">
              <template v-slot:avatar>
                <q-icon name="info" color="blue" />
              </template>
              <div class="text-weight-bold">¿Qué documento necesito?</div>
              <ul class="q-pl-md q-my-xs text-caption">
                <li>Ingrese al portal <strong>siat.impuestos.gob.bo</strong></li>
                <li>Busque su NIT y descargue el <strong>Documento de Exhibición</strong></li>
                <li>Súbalo aquí en formato PDF, JPG o PNG (máx. 5MB)</li>
                <li>Debe mostrar claramente el <strong>código QR</strong></li>
              </ul>
            </q-banner>

            <q-file v-model="nitArchivo" label="Seleccionar Documento de Exhibición NIT"
              outlined dense accept=".pdf,.jpg,.jpeg,.png,.gif,.bmp" max-file-size="5242880"
              @rejected="onNitRejected">
              <template v-slot:prepend>
                <q-icon name="upload_file" />
              </template>
            </q-file>
            <div class="text-caption text-grey-7 q-mt-xs">Formatos: PDF, JPG, PNG, GIF, BMP — Máx. 5MB</div>
          </div>

          <!-- Step 2: Processing -->
          <div v-if="nitStep === 2" class="text-center q-pa-md">
            <q-spinner-hourglass color="purple" size="48px" />
            <div class="text-subtitle2 q-mt-md">Leyendo código QR y consultando Impuestos Nacionales...</div>
            <div class="text-caption text-grey-6 q-mt-xs">Esto puede tardar unos segundos</div>
          </div>

          <!-- Step 3: Result -->
          <div v-if="nitStep === 3">
            <q-banner rounded :class="nitResultado?.estado_nit === 'VERIFICADO'
              ? 'bg-green-1 text-green-9'
              : nitResultado?.estado_nit === 'PENDIENTE'
                ? 'bg-amber-1 text-amber-9'
                : 'bg-red-1 text-red-9'">
              <template v-slot:avatar>
                <q-icon
                  :name="nitResultado?.estado_nit === 'VERIFICADO' ? 'check_circle' : nitResultado?.estado_nit === 'PENDIENTE' ? 'pending' : 'cancel'"
                  :color="nitResultado?.estado_nit === 'VERIFICADO' ? 'positive' : nitResultado?.estado_nit === 'PENDIENTE' ? 'amber-8' : 'negative'"
                  size="28px" />
              </template>
              <!-- eslint-disable-next-line vue/no-v-html -->
              <div v-html="nitMensaje"></div>
            </q-banner>
          </div>
        </q-card-section>

        <q-card-actions align="right" class="q-pb-md q-px-md">
          <q-btn v-if="nitStep !== 2" flat label="Cerrar" color="grey-7" v-close-popup
            @click="nitStep = 1; nitArchivo = null" />
          <q-btn v-if="nitStep === 1" label="Enviar para Verificación" color="purple" icon="send"
            :disable="!nitArchivo" :loading="uploadingNit" @click="uploadNitDocumento" unelevated />
          <q-btn v-if="nitStep === 3" label="Intentar de nuevo" color="purple" unelevated
            @click="nitStep = 1; nitArchivo = null" />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <!-- Dialogo de Ayuda para obtener NIT -->
    <q-dialog v-model="showNitHelpDialog">
      <q-card style="min-width: 440px; max-width: 95vw; border-radius: 12px;">
        <q-card-section class="bg-blue-8 text-white row items-center q-py-sm">
          <q-icon name="help_outline" size="24px" class="q-mr-sm" />
          <div class="text-h6">¿Cómo obtener el Documento de Exhibición?</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>
        
        <q-card-section class="q-pt-md">
          <p class="text-body2 text-grey-9 q-mb-md">Siga estos pasos para descargar su Documento de Exhibición de NIT desde el portal oficial de Impuestos Nacionales:</p>
          <q-list bordered separator class="rounded-borders q-mb-md">
            <q-item>
              <q-item-section avatar><q-avatar size="md" color="blue-1" text-color="blue-8">1</q-avatar></q-item-section>
              <q-item-section>Ingrese al <strong>SIAT en Línea</strong> e inicie sesión con sus credenciales.</q-item-section>
            </q-item>
            <q-item>
              <q-item-section avatar><q-avatar size="md" color="blue-1" text-color="blue-8">2</q-avatar></q-item-section>
              <q-item-section>En el menú principal, ingrese al módulo <strong>Padrón Biométrico Digital</strong>.</q-item-section>
            </q-item>
            <q-item>
              <q-item-section avatar><q-avatar size="md" color="blue-1" text-color="blue-8">3</q-avatar></q-item-section>
              <q-item-section>Seleccione <strong>Reimpresión de Certificados</strong> y luego <strong>Documento de Exhibición</strong>.</q-item-section>
            </q-item>
            <q-item>
              <q-item-section avatar><q-avatar size="md" color="blue-1" text-color="blue-8">4</q-avatar></q-item-section>
              <q-item-section>Descargue el archivo en formato PDF (o tome una captura de pantalla clara del código QR) y súbalo aquí.</q-item-section>
            </q-item>
          </q-list>
        </q-card-section>

        <q-card-actions align="center" class="q-pb-md">
          <q-btn outline color="primary" icon="open_in_new" label="Ir al Portal del SIAT" href="https://login.impuestos.gob.bo/" target="_blank" />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <!-- Dialogo de Rechazo de Factura -->
    <q-dialog v-model="showRejectDialog">

      <q-card style="min-width: 400px; max-width: 90vw;">
        <q-card-section class="row items-center q-pb-none">
          <div class="text-h6 text-negative row items-center">
            <q-icon name="error_outline" class="q-mr-sm" size="28px" />
            Factura Rechazada
          </div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>

        <q-card-section>
          <div class="text-body2 q-mb-md">Su factura no cumple con uno o más requisitos. Por favor revise el detalle a
            continuación:</div>

          <q-list bordered separator class="rounded-borders q-mb-md">
            <q-item v-for="(detalle, index) in rejectDetalles" :key="index"
              :class="detalle.estado === 'ok' ? 'bg-green-1' : 'bg-red-1'">
              <q-item-section avatar>
                <q-icon :name="detalle.estado === 'ok' ? 'check_circle' : 'cancel'"
                  :color="detalle.estado === 'ok' ? 'positive' : 'negative'" />
              </q-item-section>
              <q-item-section>
                <q-item-label :class="detalle.estado === 'ok' ? 'text-positive' : 'text-negative'"
                  class="text-weight-bold">{{ detalle.titulo }}</q-item-label>
                <q-item-label caption class="text-grey-9">{{ detalle.mensaje }}</q-item-label>
              </q-item-section>
            </q-item>
          </q-list>

          <q-banner rounded class="bg-orange-1 text-orange-9 q-mt-md">
            <template v-slot:avatar>
              <q-icon name="warning" color="orange" />
            </template>
            <div class="text-weight-bold">Importante:</div>
            <ul class="q-pl-md q-my-xs">
              <li><b>NO</b> suba facturas escaneadas, fotos o capturas de pantalla.</li>
              <li><b>SIEMPRE</b> use el archivo PDF original descargado del portal de Impuestos Nacionales o enviado a
                su
                correo.</li>
              <li>Si persiste el error, verifique que su archivo no esté dañado.</li>
            </ul>
          </q-banner>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn flat label="Entendido, voy a corregirlo" color="primary" v-close-popup />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script>
import { ref, onMounted } from 'vue'
import { useQuasar } from 'quasar'
import { api } from 'boot/axios'
import { useRoute, useRouter } from 'vue-router'
import jsQR from 'jsqr'

export default {
  name: 'PublicSearchPage',
  setup() {
    const $q = useQuasar()
    const route = useRoute()
    const router = useRouter()
    const ci = ref('')
    const searching = ref(false)
    const searchResult = ref(null)
    const noCorteActivo = ref(false)
    const uploading = ref({})

    // Dialog state
    const showRejectDialog = ref(false)
    const rejectDetalles = ref([])

    // NIT verification dialog state
    const showNitDialog = ref(false)
    const showNitHelpDialog = ref(false)
    const nitArchivo = ref(null)
    const uploadingNit = ref(false)
    const nitStep = ref(1)       // 1=form, 2=loading, 3=result
    const nitResultado = ref(null)
    const nitMensaje = ref('')

    const searchByCI = async (refresh = false) => {
      searching.value = true
      if (!refresh) {
        searchResult.value = null
      }
      noCorteActivo.value = false

      try {
        // Update URL query param
        router.replace({ query: { ...route.query, ci: ci.value } })

        const response = await api.get('/public/search', {
          params: { ci: ci.value }
        })
        searchResult.value = response.data

        // Add file property to each facturacion for v-model
        searchResult.value.facturaciones.forEach(f => {
          f.file = null
        })
      } catch (error) {
        if (error.response?.status === 404) {
          const message = error.response.data.message
          if (message.includes('corte activo')) {
            noCorteActivo.value = true
          } else {
            $q.notify({ type: 'warning', message })
          }
        } else {
          $q.notify({ type: 'negative', message: 'Error al buscar' })
        }
      } finally {
        searching.value = false
      }
    }

    // ─── Local QR Extraction helper using jsQR ──────────────────────────────
    const extractQRLocally = (file) => {
      return new Promise((resolve) => {
        // Only process images (skip PDF for local JS extraction)
        if (!file.type.startsWith('image/')) {
          resolve(null)
          return
        }

        const reader = new FileReader()
        reader.onload = (e) => {
          const img = new Image()
          img.onload = () => {
            const canvas = document.createElement('canvas')
            const ctx = canvas.getContext('2d')
            // Scale down slightly for performance and better jsQR parsing on large photos
            const maxDim = 1500
            let width = img.width
            let height = img.height
            if (width > maxDim || height > maxDim) {
              const ratio = Math.min(maxDim / width, maxDim / height)
              width = Math.round(width * ratio)
              height = Math.round(height * ratio)
            }
            canvas.width = width
            canvas.height = height
            ctx.drawImage(img, 0, 0, width, height)
            
            const imageData = ctx.getImageData(0, 0, width, height)
            const code = jsQR(imageData.data, imageData.width, imageData.height, {
              inversionAttempts: "dontInvert",
            })

            if (code && code.data && code.data.includes('impuestos.gob.bo')) {
              resolve(code.data)
            } else {
              resolve(null)
            }
          }
          img.onerror = () => resolve(null)
          img.src = e.target.result
        }
        reader.onerror = () => resolve(null)
        reader.readAsDataURL(file)
      })
    }

    // ─── Upload NIT Exhibicion Document ─────────────────────────────────────────
    const uploadNitDocumento = async () => {
      if (!nitArchivo.value || !searchResult.value?.docente?.id) return

      nitStep.value = 2   // Show loading spinner
      uploadingNit.value = true
      nitResultado.value = null
      nitMensaje.value = ''

      try {
        const formData = new FormData()
        formData.append('nit_documento', nitArchivo.value)

        // Intentar extraer QR en el lado del cliente (Navegador)
        const localQrUrl = await extractQRLocally(nitArchivo.value)
        if (localQrUrl) {
          formData.append('siat_url', localQrUrl)
        }

        const response = await api.post(
          `/public/docentes/${searchResult.value.docente.id}/verificar-nit`,
          formData,
          { headers: { 'Content-Type': 'multipart/form-data' } }
        )

        if (response.data.estado_nit === 'VERIFICADO' || response.data.estado_nit === 'PENDIENTE') {
          $q.notify({
            type: response.data.estado_nit === 'VERIFICADO' ? 'positive' : 'warning',
            message: response.data.message,
            html: true,
            timeout: 5000,
            position: 'top'
          })
          showNitDialog.value = false
          searchByCI(true)
          setTimeout(() => {
            nitStep.value = 1
            nitArchivo.value = null
          }, 300)
        } else {
          nitResultado.value = response.data
          nitMensaje.value = response.data.message
          nitStep.value = 3
        }

      } catch (error) {
        const errorData = error.response?.data
        nitResultado.value = { estado_nit: errorData?.estado_nit || 'INVALIDO' }
        nitMensaje.value = errorData?.message || 'Error al procesar el documento. Intente nuevamente.'
        nitStep.value = 3
      } finally {
        uploadingNit.value = false
      }
    }

    const onNitRejected = () => {
      $q.notify({ type: 'negative', message: 'El archivo debe ser PDF/imagen y no superar 5MB' })
    }

    const uploadFactura = async (facturacion) => {
      if (!facturacion.file) return

      uploading.value[facturacion.id] = true
      try {
        const formData = new FormData()
        formData.append('factura', facturacion.file)

        const response = await api.post(`/public/facturaciones/${facturacion.id}/upload`, formData, {
          headers: { 'Content-Type': 'multipart/form-data' }
        })

        // Verificar el estado de la respuesta
        const data = response.data

        if (data.estado === 'APROBADO') {
          $q.notify({
            type: 'positive',
            message: data.message,
            timeout: 5000
          })
        } else if (data.estado === 'SUBIDA' && data.validacion?.errores?.length > 0) {
          // Pendiente de revisión manual
          $q.dialog({
            title: '⚠️ Factura en Revisión',
            message: data.message + '\n\nSu factura será revisada manualmente por un administrador.',
            ok: 'Entendido'
          })
        } else {
          $q.notify({
            type: 'positive',
            message: data.message || 'Factura subida correctamente',
            timeout: 3000
          })
        }

        // Refresh search
        searchByCI(true)
      } catch (error) {
        const errorData = error.response?.data

        // Error 422 = Error de validación (factura rechazada)
        // Error 422 = Error de validación (factura rechazada)
        if (error.response?.status === 422 && errorData?.validacion) {
          // Actualizar estado local para mostrar el banner de rechazo inmediatamente
          if (errorData.facturacion) {
            const index = searchResult.value.facturaciones.findIndex(f => f.id === facturacion.id)
            if (index !== -1) {
              // Mantenemos la referencia pero actualizamos propiedades para reactividad
              Object.assign(searchResult.value.facturaciones[index], errorData.facturacion)

              // Limpiamos el archivo seleccionado
              searchResult.value.facturaciones[index].file = null
            }
          }

          rejectDetalles.value = errorData.validacion.detalles || []
          showRejectDialog.value = true
        } else {
          $q.notify({
            type: 'negative',
            message: error.response?.data?.message || 'Error al subir factura'
          })
        }
      } finally {
        uploading.value[facturacion.id] = false
      }
    }

    const confirmReplaceFactura = (facturacion) => {
      $q.dialog({
        title: 'Confirmar Reemplazo',
        message: '¿Está seguro de que desea reemplazar su factura actual? La factura anterior será eliminada y deberá esperar una nueva revisión.',
        cancel: {
          label: 'Cancelar',
          color: 'grey',
          flat: true
        },
        ok: {
          label: 'Sí, Reemplazar',
          color: 'warning',
          flat: true
        },
        persistent: true
      }).onOk(() => {
        uploadFactura(facturacion)
      })
    }

    const onRejected = () => {
      $q.notify({
        type: 'negative',
        message: 'El archivo debe ser PDF y no superar 2MB'
      })
    }

    onMounted(() => {
      if (route.query.ci) {
        ci.value = route.query.ci
        searchByCI()
      }
    })

    const formatDate = (dateString) => {
      if (!dateString) return ''
      let date
      if (dateString.includes('T')) {
        date = new Date(dateString)
      } else {
        date = new Date(dateString + 'T00:00:00')
      }
      if (isNaN(date.getTime())) return ''
      return date.toLocaleDateString('es-BO')
    }

    const formatDateDisplay = (dateString) => {
      if (!dateString) return ''
      let date
      if (typeof dateString === 'string' && dateString.includes('T')) {
        date = new Date(dateString)
      } else {
        date = new Date(dateString + 'T00:00:00')
      }
      if (isNaN(date.getTime())) return ''
      return date.toLocaleDateString('es-BO')
    }

    const getPeriodoBannerClass = () => {
      const estado = searchResult.value?.periodo_facturacion?.estado
      if (estado === 'abierto') return 'bg-green-1 text-green-9'
      if (estado === 'cerrado') return 'bg-red-1 text-red-9'
      return 'bg-grey-3 text-grey-8'
    }

    const getPeriodoIcon = () => {
      const estado = searchResult.value?.periodo_facturacion?.estado
      if (estado === 'abierto') return 'check_circle'
      if (estado === 'cerrado') return 'block'
      return 'schedule'
    }

    const getPeriodoTitle = () => {
      const estado = searchResult.value?.periodo_facturacion?.estado
      if (estado === 'abierto') return 'Periodo de Facturación Abierto'
      if (estado === 'cerrado') return 'Periodo de Facturación Cerrado'
      return 'Periodo de Facturación Pendiente'
    }

    return {
      ci,
      searching,
      searchResult,
      noCorteActivo,
      uploading,
      searchByCI,
      uploadFactura,
      confirmReplaceFactura,
      onRejected,
      formatDate,
      formatDateDisplay,
      getPeriodoBannerClass,
      getPeriodoIcon,
      getPeriodoTitle,
      showRejectDialog,
      rejectDetalles,
      // NIT verification
      showNitDialog,
      showNitHelpDialog,
      nitArchivo,
      uploadingNit,
      nitStep,
      nitResultado,
      nitMensaje,
      uploadNitDocumento,
      onNitRejected,
      viewFactura: (facturacion) => {
        if (!facturacion.factura_path) return
        // Get base URL and ensure it doesn't end with /api
        let baseUrl = api.defaults.baseURL
        if (baseUrl.endsWith('/api')) {
          baseUrl = baseUrl.slice(0, -4)
        }
        const url = `${baseUrl}/storage/${facturacion.factura_path}`
        window.open(url, '_blank')
      }
    }

  }
}
</script>
