<script setup>
import { computed, nextTick, onMounted, ref, watch } from 'vue'

const CODES = [
  'CODE128',
  'CODE128A',
  'CODE128B',
  'CODE128C',
  'EAN13',
  'EAN8',
  'UPC',
  'CODE39',
  'ITF14',
  'ITF',
  'MSI',
  'MSI10',
  'MSI11',
  'MSI1010',
  'MSI1110',
  'PHARMACODE'
]

const text = ref('Ejemplo 1234')
const selectedType = ref('code128')
const width = ref(2)
const height = ref(50)
const fontSize = ref(20)
const currentCode = ref(0)

const codes = computed(() =>
  text.value
    .split('\n')
    .map((code) => code.trim())
    .filter((code) => code.length > 0)
)

const totalPages = computed(() => codes.value.length || 1)
const currentPageLabel = computed(() => {
  if (!codes.value.length) return '0/0'
  return `${currentCode.value + 1}/${codes.value.length}`
})

function buildBarcodeOptions() {
  return {
    format: selectedType.value,
    lineColor: '#000',
    width: Number(width.value),
    height: Number(height.value),
    displayValue: true,
    fontSize: Number(fontSize.value)
  }
}

function renderCurrent() {
  const canvas = document.getElementById('canvas')
  if (!canvas) return

  if (!codes.value.length) {
    canvas.innerHTML = ''
    return
  }

  const value = codes.value[currentCode.value]
  try {
    canvas.innerHTML = ''
    window.JsBarcode(canvas, value, buildBarcodeOptions())
  } catch (error) {
    canvas.innerHTML = '<text x="50%" y="50%" dominant-baseline="middle" text-anchor="middle" fill="red">Inválido!</text>'
  }
}

function renderSvgForCode(codeValue) {
  const svg = document.createElementNS('http://www.w3.org/2000/svg', 'svg')
  svg.setAttribute('xmlns', 'http://www.w3.org/2000/svg')
  svg.setAttribute('version', '1.1')
  svg.setAttribute('width', '100%')
  svg.setAttribute('height', '100%')
  window.JsBarcode(svg, codeValue, buildBarcodeOptions())
  return new XMLSerializer().serializeToString(svg)
}

watch(
  [text, selectedType, width, height, fontSize],
  () => {
    if (!codes.value.length) {
      currentCode.value = 0
      return
    }

    if (currentCode.value >= codes.value.length) {
      currentCode.value = codes.value.length - 1
    }

    nextTick(() => renderCurrent())
  }
)

onMounted(() => {
  nextTick(() => renderCurrent())
})

function first() {
  if (!codes.value.length || currentCode.value === 0) return
  currentCode.value = 0
  renderCurrent()
}

function previous() {
  if (!codes.value.length || currentCode.value === 0) return
  currentCode.value -= 1
  renderCurrent()
}

function next() {
  if (!codes.value.length || currentCode.value >= codes.value.length - 1) return
  currentCode.value += 1
  renderCurrent()
}

function last() {
  if (!codes.value.length || currentCode.value === codes.value.length - 1) return
  currentCode.value = codes.value.length - 1
  renderCurrent()
}

function save() {
  if (!codes.value.length) return

  const zip = new window.JSZip()

  codes.value.forEach((codeValue, index) => {
    const svgContent = renderSvgForCode(codeValue)
    zip.file(`${index}_${codeValue}.svg`, svgContent)
  })

  zip.generateAsync({ type: 'blob' }).then((content) => {
    window.saveAs(content, 'barcodes.zip')
  })
}
</script>

<template>
  <div class="app-shell">
    <header class="toolbar">
      <h2>Generador de Códigos de Barras</h2>
      <svg id="canvas" class="barcode-canvas" aria-label="Barcode preview"></svg>
      <nav class="pagination" aria-label="Barcode navigation">
        <button type="button" @click="first">&lt;&lt; Primero</button>
        <button type="button" @click="previous">&lt; Anterior</button>
        <span id="page">{{ currentPageLabel }}</span>
        <button type="button" @click="next">Siguiente &gt;</button>
        <button type="button" @click="last">Último &gt;&gt;</button>
      </nav>
    </header>

    <main class="controls">
      <label for="codes">Códigos:</label>
      <textarea id="codes" v-model="text"></textarea>

      <label for="type">Tipo de Código:</label>
      <select id="type" v-model="selectedType">
        <option v-for="code in CODES" :key="code" :value="code.toLowerCase()">
          {{ code }}
        </option>
      </select>

      <label for="width">Ancho</label>
      <input id="width" v-model.number="width" type="number" step="1" min="1" />

      <label for="height">Alto:</label>
      <input id="height" v-model.number="height" type="number" step="1" min="1" />

      <label for="fontsize">Tamaño de Fuente:</label>
      <input id="fontsize" v-model.number="fontSize" type="number" step="1" min="1" />

      <button class="download-button" type="button" @click="save">Descargar</button>
    </main>
  </div>
</template>

<style scoped>
.app-shell {
  max-width: 720px;
  margin: 2rem auto;
  padding: 1.5rem;
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.06);
}

.toolbar {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

h2 {
  margin: 0;
  text-align: center;
}

.barcode-canvas {
  width: 100%;
  min-height: 120px;
  border: 1px solid #7f7f7f;
  border-radius: 8px;
  background: #fff;
}

.pagination {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 0.5rem;
}

button {
  cursor: pointer;
  border: 1px solid #bbb;
  border-radius: 6px;
  background: #f5f5f5;
  padding: 0.6rem 0.9rem;
  font-size: 0.95rem;
}

button:hover {
  background: #eaeaea;
}

.controls {
  margin-top: 1.25rem;
  display: grid;
  gap: 0.6rem;
}

textarea,
select,
input {
  width: 100%;
  box-sizing: border-box;
  padding: 0.65rem 0.75rem;
  border: 1px solid #cfcfcf;
  border-radius: 6px;
  font: inherit;
}

textarea {
  min-height: 120px;
  resize: vertical;
}

.download-button {
  width: 100%;
  margin-top: 0.5rem;
  background: #1f6feb;
  color: white;
  border-color: #1f6feb;
}

.download-button:hover {
  background: #185fcb;
}
</style>
