<script setup>
import { ref } from 'vue'
import * as XLSX from 'xlsx'
import { ArchiveOutline as ArchiveIcon } from '@vicons/ionicons5'
import {
  NButton,
  NLayout,
  NLayoutHeader,
  NLayoutContent,
  NSpace,
  NUpload,
  NUploadDragger,
  NIcon,
  NText,
  NP,
  NH1,
  NCard,
  NConfigProvider,
  darkTheme
} from 'naive-ui'

const fileList = ref([])
const invoices = ref([])

const formatNumber = (num) => Number(num || 0).toFixed(2)

const readFileContent = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()

    reader.onload = (event) => {
      try {
        const invoiceData = JSON.parse(event.target.result)
        const totalTributos = invoiceData.resumen.tributos
            ? invoiceData.resumen.tributos.reduce((sum, tributo) => {
              // Excluir el IVA (código "20") como mencionaste
              if (tributo.codigo !== "20") {
                return sum + (tributo.valor || 0);
              }
              return sum;
            }, 0) : 0;

        let formattedDate = '';
        if (invoiceData.identificacion.fecEmi) {
          const [anio, mes, dia] = invoiceData.identificacion.fecEmi.split('-');
          formattedDate = dia && mes && anio ? `${dia}/${mes}/${anio}` : invoiceData.identificacion.fecEmi;
        }

        resolve({
          formattedDate: formattedDate,
          numeroControl: invoiceData.identificacion.numeroControl || '',
          nit: invoiceData.emisor.nit || '',
          nombre: invoiceData.emisor.nombre || '',
          ventaExenta: formatNumber(totalTributos),
          subTotal: formatNumber(invoiceData.resumen.subTotal),
          iva: formatNumber(invoiceData.resumen.tributos.find(trib => trib.codigo === "20")?.valor),
          total: formatNumber(invoiceData.resumen.totalPagar)
        })
      } catch (error) {
        reject(`Error al parsear ${file.name}: ${error.message}`)
      }
    }

    reader.onerror = () => reject(`Error al leer ${file.name}`)
    reader.readAsText(file.file)
  })
}

const handleTransform = async () => {
  if (fileList.value.length === 0) {
    alert("No hay archivos seleccionados.")
    return
  }

  try {
    invoices.value = []

    for (const file of fileList.value) {
      try {
        const datosFactura = await readFileContent(file)
        invoices.value.push(datosFactura)
      } catch (error) {
        console.error(error)
      }
    }

    if (invoices.value.length > 0) {
      const worksheet = XLSX.utils.json_to_sheet(invoices.value)
      const workbook = XLSX.utils.book_new()
      XLSX.utils.book_append_sheet(workbook, worksheet, "Compras")
      XLSX.writeFile(workbook, "compras_exportadas.xlsx")
    } else {
      alert("No se pudieron procesar los archivos correctamente.")
    }
  } catch (error) {
    console.error("Error durante la transformación:", error)
    alert("Ocurrió un error durante la transformación.")
  }
}

const cleanInvoices = () => {
  invoices.value = []
  fileList.value = []
}
</script>

<template>
  <n-config-provider :theme="darkTheme">
    <n-layout style="height: 100vh">
      <n-layout-header style="padding: 24px; text-align: center">
        <n-h1>Convertidor de archivos de factura digital JSON a Excel</n-h1>
      </n-layout-header>

      <n-layout-content style="padding: 24px">
        <n-card>
          <n-space vertical align="center" size="large">
            <n-upload
                v-model:file-list="fileList"
                multiple
                directory-dnd
                :default-upload="false"
            >
              <n-upload-dragger>
                <div style="margin-bottom: 12px">
                  <n-icon size="48" :depth="3">
                    <ArchiveIcon />
                  </n-icon>
                </div>
                <n-text style="font-size: 16px">
                  Haz clic o arrastra un archivo JSON o una carpeta a esta área.
                </n-text>
                <n-p depth="3" style="margin: 8px 0 0 0">
                  Los archivos seleccionados serán procesados para su conversion a archivo Excel.
                </n-p>
              </n-upload-dragger>
            </n-upload>

            <n-space justify="center">
              <n-button type="primary" size="large" @click="handleTransform">
                Transformar
              </n-button>
              <n-button v-if="fileList.length" type="error" size="large" @click="cleanInvoices">
                Limpiar
              </n-button>
            </n-space>
          </n-space>
        </n-card>
      </n-layout-content>
    </n-layout>
  </n-config-provider>
</template>

<style scoped>

</style>
