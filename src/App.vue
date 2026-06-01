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
  {label: 'Comprobante de retencion', value: '04'},
  {label: 'Comprobante de liquidacion', value: '05'},
  {label: 'Factura sujeto excluido', value: '06'},
]

const formatNumber = (num) => Number(num || 0).toFixed(2)

const exportSettingsByDocument = {
  '01': {sheetName: 'Facturas venta', fileName: 'facturas_venta.xlsx'},
  '02': {sheetName: 'CCF venta', fileName: 'ccf_venta.xlsx'},
  '03': {sheetName: 'Compras', fileName: 'compras.xlsx'},
  '04': {sheetName: 'Comprobante retencion', fileName: 'comprobante_retencion.xlsx'},
  '05': {sheetName: 'Comprobante liquidacion', fileName: 'comprobante_liquidacion.xlsx'},
  '06': {sheetName: 'Factura sujeto excluido', fileName: 'factura_sujeto_excluido.xlsx'},
}

const readFileContent = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()

    reader.onload = (event) => {
      try {
        const invoiceData = JSON.parse(event.target.result)
        let result = {}
        const totalTributos = invoiceData.resumen?.tributos
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
            claseDocumento: '4',
            tipoDocumento: '01',
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
            claseDocumento: '4',
            tipoDocumento: '03',
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
          const ivaPerci1Value = Number(invoiceData.resumen.ivaPerci1 || 0)
          result = {
            fecha: formattedDate,
            claseDocumento: '4',
            tipoDocumento: '03',
            numeroControl: invoiceData.identificacion.numeroControl || '',
            nit: invoiceData.emisor.nit || '',
            nombre: invoiceData.emisor.nombre || '',
            ventaExenta: formatNumber(totalTributos),
            subTotal: formatNumber(invoiceData.resumen.subTotal),
            iva: formatNumber(invoiceData.resumen.tributos.find(trib => trib.codigo === "20")?.valor),
            total: formatNumber(invoiceData.resumen.totalPagar),
            ivaPercibido: formatNumber(invoiceData.resumen.ivaPerci1),
            selloRecibido: ivaPerci1Value !== 0 ? (invoiceData.selloRecibido || '') : '',
          }
        } else if (documentSelected.value === '04') {
          // Comprobante de retencion
          const cuerpoDocumento = Array.isArray(invoiceData.cuerpoDocumento) ? invoiceData.cuerpoDocumento : []
          result = cuerpoDocumento.map((item) => ({
            nit: invoiceData.emisor?.nit || '',
            fecha: formattedDate,
            tipoDocumento: '07',
            selloRecepcion: invoiceData.selloRecibido || '',
            numeroControl: invoiceData.identificacion?.numeroControl || '',
            montoSujetoRetencion: formatNumber(item.montoSujetoGrav),
            montoRetencion: formatNumber(item.ivaRetenido),
          }))
        } else if (documentSelected.value === '05') {
          // Comprobante de liquidacion
          result = {
            'Nit agente (emisor)': invoiceData.emisor?.nit || '',
            'Fecha': formattedDate,
            'Sello de recepción': invoiceData.selloRecibido || '',
            'Numero de control DTE': invoiceData.identificacion?.numeroControl || '',
            'Monto sujeto': formatNumber(invoiceData.cuerpoDocumento?.montoSujetoPercepcion),
            'Monto anticipo a cuenta 2%': formatNumber(invoiceData.cuerpoDocumento?.ivaPercibido),
          }
        } else if (documentSelected.value === '06') {
          // Factura sujeto excluido
          const numeroDocumento = invoiceData.sujetoExcluido?.numDocumento || ''
          const documentDigits = numeroDocumento.replace(/\D/g, '')
          result = {
            'Tipo de documento': documentDigits.length === 9 ? '2' : '1',
            'Numero de documento': numeroDocumento,
            'Nombre de proveedor (Receptor)': invoiceData.sujetoExcluido?.nombre || '',
            'Fecha de emisión': formattedDate,
            'Sello de recepción': invoiceData.selloRecibido || invoiceData.firmaElectronica?.selloRecibido || '',
            'Numero de documento DTE': invoiceData.identificacion?.numeroControl || '',
            'Monto de compra (Subtotal)': formatNumber(invoiceData.resumen?.subTotal),
            'Retención 10%': formatNumber(invoiceData.resumen?.reteRenta),
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
        if (Array.isArray(datosFactura)) {
          invoices.value.push(...datosFactura)
        } else {
          invoices.value.push(datosFactura)
        }
      } catch (error) {
        console.error(error)
      }
    }

    if (invoices.value.length > 0) {
      const worksheet = XLSX.utils.json_to_sheet(invoices.value)
      const workbook = XLSX.utils.book_new()
      const exportSettings = exportSettingsByDocument[documentSelected.value]
      XLSX.utils.book_append_sheet(workbook, worksheet, exportSettings.sheetName)
      XLSX.writeFile(workbook, exportSettings.fileName)
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
