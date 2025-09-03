<script setup>
import {ref} from 'vue'
import * as XLSX from 'xlsx'
import {ArchiveOutline as ArchiveIcon} from '@vicons/ionicons5'
import {
  NButton,
  NLayout,
  NLayoutHeader,
  NLayoutContent,
  NSelect,
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
const documentSelected = ref(null)
const options = [
  {label: 'Factura de Venta', value: '01'},
  {label: 'Comprobante de credito fiscal de Venta', value: '02'},
  {label: 'Compras', value: '03'},
]

const formatNumber = (num) => Number(num || 0).toFixed(2)

const readFileContent = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()

    reader.onload = (event) => {
      try {
        const invoiceData = JSON.parse(event.target.result)
        let result = {}
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

        if (documentSelected.value === '01') {
          // Factura de venta
          result = {
            fecha: formattedDate,
            codigoGeneracion: invoiceData.identificacion.codigoGeneracion || '',
            selloRecibido: invoiceData.selloRecibido || '',
            numeroControl: invoiceData.identificacion.numeroControl || '',
            numeroControlCopia: invoiceData.identificacion.numeroControl || '',
            numeroControlCopia2: invoiceData.identificacion.numeroControl || '',
            numeroControlCopia3: invoiceData.identificacion.numeroControl || '',
            total: formatNumber(invoiceData.resumen.totalPagar),
            totalCopia: formatNumber(invoiceData.resumen.totalPagar)
          }
        } else if (documentSelected.value === '02') {
          // CCF de venta
          result = {
            fecha: formattedDate,
            codigoGeneracion: invoiceData.identificacion.codigoGeneracion || '',
            selloRecibido: invoiceData.selloRecibido || '',
            numeroControl: invoiceData.identificacion.numeroControl || '',
            numeroControlCopia: invoiceData.identificacion.numeroControl || '',
            nit: invoiceData.receptor.nit || '',
            nombre: invoiceData.receptor.nombre || '',
            subTotal: formatNumber(invoiceData.resumen.subTotal),
            iva: formatNumber(invoiceData.resumen.tributos.find(trib => trib.codigo === "20")?.valor),
            total: formatNumber(invoiceData.resumen.totalPagar)
          }
        } else if (documentSelected.value === '03') {
          // Compras
          result = {
            fecha: formattedDate,
            numeroControl: invoiceData.identificacion.numeroControl || '',
            nit: invoiceData.emisor.nit || '',
            nombre: invoiceData.emisor.nombre || '',
            ventaExenta: formatNumber(totalTributos),
            subTotal: formatNumber(invoiceData.resumen.subTotal),
            iva: formatNumber(invoiceData.resumen.tributos.find(trib => trib.codigo === "20")?.valor),
            total: formatNumber(invoiceData.resumen.totalPagar)
          }
        }

        resolve(result)
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

  if (!documentSelected.value) {
    alert("Por favor, seleccione el tipo de documento.")
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
      const fileName = documentSelected.value === '01' ? 'facturas_venta.xlsx' :
        documentSelected.value === '02' ? 'ccf_venta.xlsx' : 'compras.xlsx'
      XLSX.writeFile(workbook, fileName)
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

      <n-layout-content style="padding: 24px; max-width: 800px; margin: 0 auto">
        <n-card>
          <n-space vertical size="large">
            <n-select
                v-model:value="documentSelected"
                placeholder="Seleccione el documento"
                size="large"
                :options="options"
            />

            <n-upload
                v-model:file-list="fileList"
                file-list-style="max-height: 300px; overflow-y: auto;"
                multiple
                directory-dnd
                :default-upload="false"
            >
              <n-upload-dragger>
                <div style="margin-bottom: 12px">
                  <n-icon size="48" :depth="3">
                    <ArchiveIcon/>
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
