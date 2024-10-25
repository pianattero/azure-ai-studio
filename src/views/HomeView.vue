<template>
  <main class="main-container">
    <TheSideMenu />
    <div v-if="!showCertificateChecks" class="content-container">
      <div class="doc-uploader-wrapper">
        <label class="upload-wrapper" :class="{ 'disabled-button': loading }">
          <div v-if="!loading">
            Cargar Archivo
            <input
              id="fileInput"
              type="file"
              accept=".pdf, .png, .jpg, .jpeg"
              name="file-reader"
              class="file-reader"
              @change="onFileChange"
            />
          </div>
          <img v-else class="spin" src="@/assets/icons/sync.svg" />
        </label>

        <div
          v-if="documents.length > 0 && certificatePoCValidation?.legibilityStatus && !loading"
          class="docs-wrapper"
        >
          <div v-for="(file, index) in documents" :key="index" class="doc-attached">
            <span>{{ file.name }}</span>
            <div @click="handleOpen" class="item-action">
              <IconEye />
              <span>Ver Documento</span>
            </div>
          </div>
        </div>
        <div v-else-if="loading" class="docs-wrapper">
          <span>Cargando...</span>
        </div>
        <div v-else class="docs-wrapper">
          <span>No hay documentos adjuntos</span>
        </div>
      </div>

      <span
        v-if="certificatePoCValidation?.legibilityStatus === false"
        class="legibility-warning"
        >{{ certificatePoCValidation.message }}</span
      >
    </div>
    <CertificateChecks v-if="showCertificateChecks" :file-url="fileData" img-name="imgName" />
  </main>
</template>

<script setup lang="ts">
import { ref, type Ref } from 'vue'
import { storeToRefs } from 'pinia'
import { PoCService } from '@/service/PoCService'
import { useCertificateStore } from '@/stores/certificateStore'
import IconEye from '@/components/IconEye.vue'
import TheSideMenu from '@/components/TheSideMenu.vue'
import CertificateChecks from '@/components/CertificateChecks.vue'

// STORE
const { showCertificateChecks, certificatePoCValidation, isDocInfoExtacted } =
  storeToRefs(useCertificateStore())

// SERVICE
const { getCertificateValidation } = PoCService.getInstance()

// DATA
const documents: Ref<File[]> = ref([])
const uploadFile = ref<File | null>(null)
const fileData = ref<string | null>(null)
const loading: Ref<boolean> = ref(false)

// METHODS
function createFormData() {
  const formData = new FormData()
  if (uploadFile.value) formData.append('file', uploadFile.value)

  return formData
}

async function onFileChange(event: Event) {
  loading.value = true

  const target = event.target as HTMLInputElement

  if (target.files && target.files.length > 0) {
    uploadFile.value = target.files[0]
    const reader = new FileReader()

    documents.value.length === 0
      ? documents.value.push(uploadFile.value)
      : (documents.value[0] = uploadFile.value)

    // console.log('DOCS', documents.value[0])

    reader.onload = () => {
      // Guardar el archivo como base64 en localStorage
      localStorage.setItem('uploadedFile', reader.result as string)
      fileData.value = reader.result as string // Solo para mostrar imágenes
    }

    reader.readAsDataURL(uploadFile.value)

    const formData = createFormData()

    await getCertificateValidation(formData)
      .then((res) => {
        if (res) {
          console.log('RES', res.data)
          isDocInfoExtacted.value = false
          certificatePoCValidation.value = res.data
        }
      })
      .catch((err) => {
        console.log('ERR', err)
      })
      .finally(() => {
        loading.value = false
      })
  }
}

function handleOpen() {
  showCertificateChecks.value = true
}
</script>

<style scoped lang="scss">
.main-container {
  display: flex;

  & .content-container {
    width: calc(100vw - $sidemenu-size);
    display: flex;
    flex-direction: column;
    justify-content: center;
    gap: 0.5em;

    & .legibility-warning {
      color: $c-error;
      font-weight: bold;
    }

    & .doc-uploader-wrapper {
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 2em;

      & .upload-wrapper {
        display: flex;
        align-items: center;
        justify-content: center;
        min-width: 170px;
        height: 3rem;
        border: none;
        position: relative;
        color: $c-white;
        font-size: 16px;
        background-color: $c-turquoise-80;
        border-radius: 8px;
        cursor: pointer;
        font-family: $roboto-400;
        padding: 0 1.2rem;

        & .spin {
          animation: spin linear 0.5s infinite;
        }
      }

      & .upload-wrapper input {
        display: none;
      }

      & .disabled-button {
        background: $c-grey-30;
        border-color: $c-grey-30;
        color: $c-grey-50;
        pointer-events: none;
        cursor: auto;
      }

      & .docs-wrapper {
        display: flex;
        flex-direction: column;
        align-items: flex-start;
        gap: 0.5em;
        border: $c-grey-30 1px solid;
        border-radius: 8px;
        padding: 1em;
        width: 30em;

        & .doc-attached {
          width: 100%;
          display: flex;
          align-items: center;
          justify-content: space-between;
        }

        & .item-action {
          display: flex;
          align-items: center;
          cursor: pointer;
          color: $c-turquoise-80;

          svg {
            margin-right: 0.5rem;
          }
        }
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
