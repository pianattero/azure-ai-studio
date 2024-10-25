<template>
  <div class="page-wrapper">
    <div class="document-detail">
      <!-- IMAGEN DEL DOCUMENTO -->
      <div class="image-wrapper">
        <figure
          v-if="!originalImg.startsWith('data:application/pdf')"
          ref="zoomImg"
          class="zoom-image"
          :style="[`background-image: ${backgroundImgCss}`]"
        >
          <img :src="rotatedImg" class="static-image" />
        </figure>
        <div class="zoom-buttons">
          <button @click="zoomOut()">-</button>
          <button @click="zoomIn()">+</button>
        </div>
        <iframe
          id="iframe"
          :key="iframeKey"
          class="pdf-image"
          :src="iframeSrc"
          frameborder="0"
          allow="fullscreen"
        >
        </iframe>
      </div>

      <!-- HEADER CERTIFICADO -->
      <div class="validation-wrapper">
        <div class="doc-number">
          <h3 class="title">
            <u>
              {{
                certificatePoCValidation?.docType === docTypeEnum.ADR
                  ? 'Certificado ADR'
                  : 'Certificado de Lavado'
              }}
            </u>
          </h3>
          <div class="detail-header-icons">
            <img class="icon-close" src="@/assets/icons/close.svg" @click="handleClose" />
          </div>
        </div>

        <!-- CERTIFICADO DE LAVADO -->
        <div
          v-if="certificatePoCValidation?.docType === docTypeEnum.CertificadoLavado"
          class="section-container"
        >
          <div class="section-wrapper">
            <ul class="items-list">
              <li class="item" v-for="item in washingCertificateData" :key="item.field">
                <b>{{ item.field }}:</b>
                {{
                  item.key === 'Conductor'
                    ? cleanName(certificatePoCValidation?.extractedFields[item.key]?.toUpperCase())
                    : certificatePoCValidation?.extractedFields[item.key]?.toUpperCase()
                }}
              </li>
            </ul>
          </div>
          <div
            class="section-wrapper"
            v-for="(section, index) in currentCertificate.sections"
            :key="index"
          >
            <span class="section-title">{{ section.title }}</span>
            <div class="validation-list">
              <AppCheckbox
                v-for="(item, index) in section.validationPoints"
                class="checkbox"
                :key="index"
                :id="item.id"
                :title="item.title"
                :validated="item.validated"
                :required="item.required"
                :errors="errors"
                @click="checkValidationPoint(item.id, section.id)"
              />
            </div>
          </div>
          <div class="prohibited-wrapper">
            <span class="section-title">Códigos prohibidos</span>
            <ul class="items-list">
              <li
                class="item"
                v-for="(item, index) in currentCertificate.prohibitedCodes"
                :key="index"
              >
                {{ item }}
              </li>
            </ul>
          </div>
          <span class="error-text" v-if="errors">{{ errorMsg }}</span>
          <div class="buttons">
            <button class="button" @click="handleClose">Rechazar documento</button>
            <button class="button" @click="handleValidationCheck">Validar documento</button>
          </div>
        </div>

        <!-- CERTIFICADO ADR -->
        <div v-if="certificatePoCValidation?.docType === docTypeEnum.ADR" class="section-container">
          <div class="export-button">
            <button
              class="button"
              :class="{ 'disabled-button': isDocInfoExtacted || visibleSections[0] }"
              v-if="!loadingDocInfo"
              @click="exportInfo"
            >
              Extraer datos
            </button>
            <button class="button disabled-button" v-else @click="exportInfo">
              <img class="spin" src="@/assets/icons/sync.svg" />
            </button>
          </div>

          <div class="content-wrapper">
            <div
              class="section-wrapper"
              :class="{ visible: visibleSections[0], invisible: !isDocInfoExtacted }"
            >
              <ul class="items-list">
                <li class="item" v-for="item in adrCertificateData.section1" :key="item.key">
                  <b>{{ item.field }}:</b>
                  {{ certificatePoCValidation.extractedFields[item.key] }}
                </li>
              </ul>
            </div>
            <div
              class="section-wrapper"
              :class="{ visible: visibleSections[1], invisible: !isDocInfoExtacted }"
            >
              <span class="section-title"><b>Descripción del vehículo:</b></span>
              <ul class="items-list">
                <li class="item" v-for="item in adrCertificateData.section3" :key="item.key">
                  {{ certificatePoCValidation.extractedFields[item.key] }}
                </li>
              </ul>
            </div>
            <div
              class="section-wrapper"
              :class="{ visible: visibleSections[2], invisible: !isDocInfoExtacted }"
            >
              <span class="section-title"><b>Dispositivo de frenos de resistencia:</b></span>
              <ul class="items-list">
                <AppCheckbox
                  class="checkbox"
                  :title="adrCertificateData.section5[0].field"
                  :disabled="true"
                  :validated="
                    certificatePoCValidation.extractedFields[adrCertificateData.section5[0].key] ===
                    ':selected:'
                      ? true
                      : false
                  "
                />
                <AppCheckbox
                  class="checkbox"
                  :title="`${adrCertificateData.section5[1].field} 
                    ${certificatePoCValidation.extractedFields[adrCertificateData.section5[2].key] ? certificatePoCValidation.extractedFields[adrCertificateData.section5[2].key] : '________________'} 
                    t`"
                  :disabled="true"
                  :validated="
                    certificatePoCValidation.extractedFields[adrCertificateData.section5[1].key] ===
                    ':selected:'
                      ? true
                      : false
                  "
                />
              </ul>
            </div>
            <div
              class="section-wrapper"
              :class="{ visible: visibleSections[3], invisible: !isDocInfoExtacted }"
            >
              <span class="section-title"
                ><b>Mercancías peligrosas autorizadas para su transporte:</b></span
              >
              <ul class="items-list">
                <AppCheckbox
                  v-for="item in adrCertificateData.section7"
                  :key="item.key"
                  class="checkbox"
                  :title="item.field"
                  :disabled="true"
                  :validated="
                    certificatePoCValidation.extractedFields[item.key] === ':selected:'
                      ? true
                      : false
                  "
                />
              </ul>
            </div>
            <div v-if="isDocInfoExtacted" class="buttons">
              <button class="button" @click="handleClose">Rechazar documento</button>
              <button class="button" @click="handleClose">Validar documento</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
// IMPORTS
import { storeToRefs } from 'pinia'
import { computed, onMounted, ref, watch, type ComputedRef, type Ref } from 'vue'
import { useCertificateStore } from '@/stores/certificateStore'
import AppCheckbox from './AppCheckbox.vue'

// STORE
const { showCertificateChecks, currentCertificate, certificatePoCValidation, isDocInfoExtacted } =
  storeToRefs(useCertificateStore())

// TYPES
interface ValidationPoint {
  title: string
  isValidated: boolean
  required: boolean
}

interface ExtractedFieldPoint {
  extractedFieldPoint: string
  value: string | null
}

interface ProhibitedExtractedFieldPoint {
  extractedProhibitedPoint: string
  value: string | null
}

interface ProhibitedPoints {
  pocPoint: ProhibitedExtractedFieldPoint
}

interface AllPoints {
  manualVal: ValidationPoint
  pocPoint: ExtractedFieldPoint
}

enum docTypeEnum {
  CertificadoLavado = 'CertificadoLavado',
  ADR = 'ADR'
}

// PROPS
const props = defineProps({
  fileUrl: {
    type: [String, null],
    required: true
  }
})

// DATA
const originalImg: Ref<string> = ref('')
const rotatedImg: Ref<string> = ref('')
const zoomImg: Ref<HTMLElement> = ref({} as HTMLElement)
const zoomPdf: Ref<number> = ref(75)
const iframeKey: Ref<number> = ref(0)

const errors: Ref<string[]> = ref([])
const errorMsg: Ref<string> = ref('')

const visibleSections: Ref<boolean[]> = ref([
  isDocInfoExtacted.value,
  isDocInfoExtacted.value,
  isDocInfoExtacted.value,
  isDocInfoExtacted.value
])
const loadingDocInfo: Ref<boolean> = ref(false)
const PoCValidationDone: Ref<boolean> = ref(false)
const washingCertificateCodes = [
  'C01',
  'C10',
  'C20',
  'E35',
  'E41',
  'E50',
  'E51',
  'E52',
  'E55',
  'E56',
  'E57',
  'E58',
  'E60',
  'E61',
  'E62',
  'E63',
  'E64',
  'E71',
  'E72',
  'E75',
  'E77',
  'E78',
  'E79',
  'E90',
  'F01',
  'F50',
  'P01',
  'P10',
  'P30',
  'P40',
  'W50'
]
const prohibitedCodes = ['C01', 'C10', 'C20', 'F51']
const washingCertificateData = [
  { field: 'ET', key: 'Nombre' },
  { field: 'Conductor', key: 'Conductor' },
  { field: 'Producto', key: 'Producto' },
  { field: 'Matrícula Vehículo', key: 'Vehículo' },
  { field: 'Matrícula Cisterna/Contenedor', key: 'Cisterna/Contenedor' }
]

const adrCertificateData = {
  section1: [
    { field: 'Certificado Nº', key: 'Numero_certificado' },
    { field: 'Constructor del Vehículo', key: 'Constructor_vehiculo' },
    { field: 'Nº Identificación del Vehículo', key: 'Identificacion_vehiculo' },
    { field: 'Nº de Matrícula', key: 'Matricula' }
  ],
  section2: [
    { field: 'Nombre de la sede social del transportista, usuario o propietario', key: 'Nombre' },
    {
      field: 'Domicilio de la sede social del transportista, usuario o propietario',
      key: 'Domicilio'
    }
  ],
  section3: [{ field: 'Descripción del vehículo', key: 'Descripcion_vehiculo' }],
  section4: [
    {
      field: 'Designación(es) del vehículo según el 9.1.1.2 del ADR',
      key: 'Designaciones_vehiculo'
    }
  ],
  section5: [
    {
      field: 'No aplicable',
      key: 'Disp_frenos_resistencia_no_aplicable'
    },
    {
      field:
        'La eficacia según 9.2.3.1.2 del ADR es suficiente para un peso total de la unidad de transporte de',
      key: 'Disp_frenos_resistencia_eficacia_suficiente'
    },
    { field: 'Peso total', key: 'Peso_total' }
  ],
  section6: [
    {
      field: 'Descripción de la cisterna fija/vehículo batería (si procede)',
      key: 'Descripción_cisterna_fija_vehiculo_bateria'
    }
  ],
  section7: [
    {
      field: 'Mercancías de la clase 1, incluyendo el grupo de compatibilidad J',
      key: 'Mercancias_clase_1_incl_grupo_comp_j'
    },
    {
      field: 'Mercancías de la clase 1, exceptuando el grupo de compatibilidad J',
      key: 'Mercancias_clase_1_excep_grupo_comp_j'
    },
    {
      field:
        'Solamente se podrán transportar (5) las materias autorizadas de acuerdo con el código de cisterna y cualquier disposición especial indicada en el Nº9',
      key: 'Solo_trans_materias_autorizadas'
    },
    {
      field:
        'Solamente se podrán transportar las materias siguientes (Clase, Nº ONU, y si fuera necesario el grupo de embalaje y la designación oficial de transporte): Este certificado se complementa con un listado de 8 Mercancías Peligrosas.',
      key: 'Solo_trans_materias_siguientes'
    }
  ],
  section8: [
    { field: 'Observaciones', key: 'Observaciones' },
    { field: 'Válido hasta', key: 'Valido_hasta' }
  ],
  section9: [
    { field: 'Validez prorrogada hasta', key: 'Validez_prorrogada_hasta_1' },
    { field: 'Validez prorrogada hasta', key: 'Validez_prorrogada_hasta_2' },
    { field: 'Validez prorrogada hasta', key: 'Validez_prorrogada_hasta_3' },
    { field: 'Validez prorrogada hasta', key: 'Validez_prorrogada_hasta_4' },
    { field: 'Validez prorrogada hasta', key: 'Validez_prorrogada_hasta_5' },
    { field: 'Observaciones: (Continuación)', key: 'Observaciones_continuacion' }
  ]
}

// HOOKS
onMounted(() => {
  if (props.fileUrl) {
    originalImg.value = props.fileUrl
    rotatedImg.value = props.fileUrl
  }

  currentCertificate.value.sections.forEach((section) => {
    section.validationPoints.forEach((validationPoint) => {
      validationPoint.validated = false
    })
  })
})

// WATCHERS
watch(originalImg, async (newImg, oldImg) => {
  if (newImg) {
    createZoomEvent()
  }
})

// COMPUTED
const backgroundImgCss: ComputedRef<string> = computed(() => `url(${rotatedImg.value})`)
const iframeSrc = computed(() => {
  return originalImg.value + `#toolbar=0&navpanes=0&zoom=${zoomPdf.value}`
})

// METHODS
function handleClose() {
  showCertificateChecks.value = false
}

function showSectionsInfo() {
  let index = 0

  function showSection() {
    if (index < visibleSections.value.length) {
      visibleSections.value[index] = true
      index++
      setTimeout(showSection, 1000)
    } else {
      isDocInfoExtacted.value = true
    }
  }

  showSection()
}

function exportInfo() {
  loadingDocInfo.value = true

  setTimeout(() => {
    loadingDocInfo.value = false
    showSectionsInfo()
  }, 2000)
}

function zoomIn() {
  zoomPdf.value += 25
  iframeKey.value++
}

function zoomOut() {
  if (zoomPdf.value > 25) zoomPdf.value -= 25
  iframeKey.value++
}

function cleanName(name: string | undefined): string | undefined {
  const prefixs = ['NAP', 'NAM', 'NAME']

  if (!name) return

  for (const prefix of prefixs) {
    if (name.startsWith(prefix)) {
      return name.slice(prefix.length)
    }
  }

  return name
}

function checkValidationPoint(id: number, sectionId: number) {
  let validationPoint = currentCertificate.value.sections
    .find((item) => item.id === sectionId)
    ?.validationPoints.find((item) => item.id === id)

  if (validationPoint) validationPoint.validated = !validationPoint.validated
}

function createZoomEvent() {
  zoomImg.value?.addEventListener('mousemove', function (event: MouseEvent) {
    const targetElement = event.currentTarget as HTMLImageElement
    if (targetElement) {
      let x = (event.offsetX / targetElement.offsetWidth) * 100
      let y = (event.offsetY / targetElement.offsetHeight) * 100
      targetElement.style.backgroundPosition = x + '% ' + y + '%'
    }
  })
}

function getValidatedChecks() {
  let pointArr: ValidationPoint[] = []
  currentCertificate.value.sections.forEach((section) => {
    section.validationPoints.forEach((point) => {
      pointArr.push({ title: point.title, isValidated: point.validated, required: point.required })
    })
  })

  return pointArr
}

function handleValidationCheck() {
  if (!PoCValidationDone.value) {
    PoCValidationDone.value = true

    getProhibitedCodes()
    getValidationPoints()

    if (errors.value.length > 0) {
      errorMsg.value =
        'Se identificaron uno o más códigos prohibidos y/o algunas diferencias entre el documento y la validación. Por favor, te sugerimos revisarlo nuevamente para evitar posibles errores.'
    }
  } else handleClose()
}

function getProhibitedCodes() {
  let prohibitedPointsArray: any = []

  prohibitedCodes.forEach((code) => {
    const extractedProhibitedPoint = Object.keys(
      certificatePoCValidation.value!.extractedFields
    ).find((prohibitedCode) => prohibitedCode.includes(code))

    if (extractedProhibitedPoint) {
      const value = certificatePoCValidation.value?.extractedFields[extractedProhibitedPoint]
      prohibitedPointsArray.push({
        pocPoint: { extractedProhibitedPoint, value }
      })
    }
  })

  prohibitedPointsArray.forEach((prohibitedPoint: ProhibitedPoints) => {
    if (!prohibitedPoint.pocPoint.value) return
    else errors.value.push(prohibitedPoint.pocPoint.extractedProhibitedPoint)
  })
}

function getValidationPoints() {
  const validationPoints = getValidatedChecks()
  let validationPointsArray: any = []

  washingCertificateCodes.forEach((code) => {
    const validationPoint: ValidationPoint | undefined = validationPoints.find((point) =>
      point.title.includes(code)
    )
    const extractedFieldPoint = Object.keys(certificatePoCValidation.value!.extractedFields).find(
      (key) => key.includes(code)
    )

    if (validationPoint && extractedFieldPoint) {
      const value = certificatePoCValidation.value?.extractedFields[extractedFieldPoint]
      validationPointsArray.push({
        manualVal: validationPoint,
        pocPoint: { extractedFieldPoint, value }
      })
    }
  })

  validationPointsArray.forEach((points: AllPoints) => {
    if (points.pocPoint.value) {
      if (points.manualVal.isValidated) return
      else errors.value.push(points.manualVal.title)
    } else if (!points.pocPoint.value) {
      if (!points.manualVal.isValidated) return
      else errors.value.push(points.manualVal.title)
    }
  })
}
</script>

<style lang="scss" scoped>
.page-wrapper {
  background-color: $c-grey-10;
  padding: 2rem 1rem;
  margin: 1rem 12rem;

  .document-detail {
    display: flex;
    gap: 2rem;
    margin-top: 1rem;
    height: calc(100vh - 12rem);
    min-width: calc(100vw - 30rem);
    overflow-y: auto;
    padding-bottom: 1rem;

    .image-wrapper {
      position: relative;
      width: 45%;

      .zoom-buttons {
        display: flex;
        justify-content: center;
        align-items: center;
        gap: 1rem;
        margin-bottom: 1rem;

        & button {
          display: flex;
          justify-content: center;
          align-items: center;
          width: 20px;
          height: 2rem;
          border: none;
          cursor: pointer;
          border-radius: 8px;
          padding: 0 1.2rem;
          font-weight: bold;
          background-color: $c-turquoise-20;
        }
      }

      .pdf-image {
        width: 100%;
        height: 45rem;
      }

      .zoom-image {
        cursor: zoom-in;
        position: absolute;
        overflow: hidden;
        background-size: 150%;

        & .static-image {
          transition: opacity 0.5s;
          width: 100%;
          &:hover {
            opacity: 0;
          }
        }
      }
    }

    .validation-wrapper {
      display: flex;
      flex-direction: column;
      width: 55%;
      margin-right: 1rem;
      padding-bottom: 1rem;

      .doc-number {
        display: flex;
        justify-content: space-between;
        h3 {
          font-size: 1.5rem;
          margin-bottom: 1.5rem;
        }
      }

      .button {
        cursor: pointer;
        margin-bottom: 1em;
        width: 200px;
        height: 3rem;
        border: none;
        font-size: 16px;
        border-radius: 8px;
        padding: 0 1.2rem;
        color: $c-white;
        background-color: $c-turquoise-80;

        & .spin {
          margin-top: 4px;
          animation: spin linear 0.5s infinite;
        }
      }

      .disabled-button {
        background: $c-grey-30;
        border-color: $c-grey-30;
        color: $c-grey-50;
        pointer-events: none;
        cursor: auto;
      }

      .content-wrapper {
        & .invisible {
          opacity: 0;
          transition: opacity 1s ease-in-out;
        }

        & .visible {
          opacity: 1;
        }
      }

      .section-container {
        display: flex;
        flex-direction: column;

        & .export-button {
          display: flex;
          justify-content: center;
        }

        & .buttons {
          display: flex;
          align-items: center;
          justify-content: end;
          gap: 2rem;
          margin-top: 2rem;

          .button:first-child {
            background-color: $c-turquoise-invent;
            color: $c-turquoise-80;
          }

          // .button:last-child {
          //   color: $c-white;
          //   background-color: $c-turquoise-80;
          // }
        }

        .section-wrapper {
          text-align: left;
          display: flex;
          flex-direction: column;
          margin-bottom: 1.5rem;

          .section-title {
            font-size: 16px;
            line-height: 24px;
            margin-bottom: 12px;
          }

          .validation-list {
            background-color: $c-white;
            display: flex;
            flex-direction: column;
            padding: 1rem 1.5rem;

            .checkbox {
              margin: 0.5rem 1rem 0.5rem 0;
              text-align: left;
              line-height: 1.8;
              font-size: 16px;
            }
          }
        }

        .prohibited-wrapper {
          text-align: left;
        }

        .items-list {
          background-color: $c-white;
          display: flex;
          flex-direction: column;
          padding: 1rem 1.5rem;
          list-style: disc;

          .item {
            margin: 0.3rem 0.7rem;
          }

          & label {
            margin-bottom: 10px;
          }
        }

        .error-text {
          text-align: left;
          color: $c-error;
          font-weight: bold;
          font-size: 16px;
        }
      }

      .section-container:last-child {
        gap: 0.5rem;
      }

      .icon-close {
        cursor: pointer;
      }
    }
  }
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}
</style>
