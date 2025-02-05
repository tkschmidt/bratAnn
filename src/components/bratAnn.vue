<template>
  <div class="brat-annotation">
    <div class="annotation-input">
      <textarea 
        v-model="inputText" 
        placeholder="Paste your BRAT annotations here..."
        class="text-input"
        @input="handleTextChange"
      ></textarea>
      <div class="parsed-output">
        <p>Parsed Annotations:</p>
        <table class="annotation-table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Type</th>
              <th>Start</th>
              <th>Stop</th>
              <th>Value</th>
              <th>Meta</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="ann in parsedAnnotations" :key="ann.id">
              <td>{{ ann.id }}</td>
              <td :style="getTypeStyle(ann.key)">{{ ann.key }}</td>
              <td>{{ ann.start_position }}</td>
              <td>{{ ann.stop_position }}</td>
              <td>{{ ann.value }}</td>
              <td>{{ ann.meta || '-' }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <div class="text-visualization">
      <h3>Text Visualization</h3>
      <AnnotatedText 
        :annotations="parsedAnnotations"
        :default-text="defaultText"
      />
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import AnnotatedText from './AnnotatedText.vue'
import chroma from 'chroma-js'

const defaultText = `Sample preparation for mass spectrometry. Dried peptide pools
were initially solubilized in 100% DMSO to a concentration of
10 pmol/µl by vortexing for 30 min at room temperature. The
pools were then diluted to 10% DMSO using 1% formic acid in
HPLCgrade water to a stock solution concentration of 1 pmol/µl
and stored at −20 °C until use. 10 µl of the stock solution was
transferred to a 96well plate and spiked with two retention
time (RT) standards. The first set of RT peptides (JPT Peptide
Technologies) consisted of 66 peptides with nonnaturally occurring peptide sequences (Supplementary Table 1). 200 fmol
per peptide was used per injection. The second RT standard
(Pierce, Thermo Scientific) comprised 15 13Clabeled peptides, and 100 fmol per peptide was used per injection. Samples
in the resulting 96well plates were vacuum dried and stored
at −20 °C until use`

const defaultAnnotations = `T1	ConcentrationOfCompound	85	101	10 pmol/µl	Explicit
T2	Time	130	136	30 min	Explicit
T3	Temperature	140	154	room temperature	Explicit
T4	Compound	67	71	DMSO	Explicit
T5	Compound	203	215	formic acid	Explicit
T6	ConcentrationOfCompound	177	189	10% DMSO	Explicit
T7	ConcentrationOfCompound	241	250	1 pmol/µl	Explicit
T8	Temperature	275	280	-20 °C	Explicit
T9	Volume	292	296	10 µl	Explicit
T10	SampleTreatment	343	356	spiked	Explicit
T11	SpikedCompound	359	379	retention time (RT) standards	Explicit
T12	SpikedCompound	405	432	RT peptides (JPT Peptide Technologies)	Explicit
T13	SyntheticPeptide	448	457	66 peptides	Explicit
T14	ConcentrationOfCompound	511	521	200 fmol	Explicit
T15	SpikedCompound	569	593	RT standard (Pierce, Thermo Scientific)	Explicit
T16	SyntheticPeptide	606	617	15 13C-labeled peptides	Explicit
T17	ConcentrationOfCompound	623	632	100 fmol	Explicit
T18	Temperature	720	725	-20 °C	Explicit`

const inputText = ref(defaultAnnotations)
const parsedAnnotations = ref([])

const annotationsWithLength = computed(() => {
  return parsedAnnotations.value.map(ann => {
    const valueLength = ann.value.length
    const spanLength = ann.stop_position - ann.start_position
    const hasLengthMismatch = valueLength !== spanLength
    const lengthDelta = valueLength - spanLength
    
    return {
      ...ann,
      valueLength,
      hasLengthMismatch,
      lengthDelta,
      mismatchInfo: hasLengthMismatch 
        ? `Mismatch: Value length (${valueLength}) ≠ Span length (${spanLength}), Delta: ${lengthDelta}`
        : ''
    }
  })
})

const normalizeText = (text) => {
  return text
    .replace(/−/g, '-')     // Replace minus sign with hyphen-minus
    .replace(/['']/g, "'")  // Normalize single quotes
    .replace(/[""]/g, '"')  // Normalize double quotes
    .replace(/\u2009/g, ' ') // Replace thin spaces
    .replace(/µ/g, 'µ')     // Normalize micro symbol
    .replace(/\r\n/g, '\n') // Normalize line endings
    .replace(/(\d+)\s*[°˚]\s*([CF])/g, '$1°$2')  // Normalize temperature format
    .replace(/[−‐‑‒–—―]/g, '-') // Normalize all types of dashes/hyphens
}

const handleTextChange = () => {
  const lines = inputText.value.split('\n').filter(line => line.trim())
  
  parsedAnnotations.value = lines.map(line => {
    const parts = line.split('\t')
    return {
      id: parts[0] || '',
      key: parts[1] || '',
      start_position: parseInt(parts[2]) || 0,
      stop_position: parseInt(parts[3]) || 0,
      value: parts[4] ? normalizeText(parts[4]) : '',
      meta: parts[5] || null
    }
  })
}

// Initialize with default annotations
handleTextChange()

// Get unique keys from annotations
const uniqueKeys = computed(() => {
  return [...new Set(parsedAnnotations.value.map(ann => ann.key))].sort()
})

// Generate color map using chroma-js
const colorMap = computed(() => {
  const keys = uniqueKeys.value
  if (keys.length === 0) return new Map()

  let colors
  if (keys.length <= 8) {
    colors = chroma.brewer.Set2
  } else if (keys.length <= 12) {
    colors = chroma.brewer.Set3
  } else {
    colors = chroma
      .scale(['#00429d', '#93003a', '#35b779', '#ff8e6d', '#ffd700', '#93003a'])
      .mode('lch')
      .colors(keys.length)
  }
  
  return new Map(keys.map((key, index) => [
    key, 
    chroma(colors[index % colors.length]).alpha(0.2).css()
  ]))
})

const getTypeStyle = (type) => {
  return {
    backgroundColor: colorMap.value.get(type) || 'rgba(200, 200, 200, 0.2)'
  }
}
</script>

<style scoped>
.brat-annotation {
  width: 100%;
  max-width: 100%;
  padding: 1rem;
  box-sizing: border-box;
}

.annotation-input, .text-visualization {
  margin-bottom: 2rem;
  width: 100%;
  max-width: 100%;
}

.text-input {
  width: 100%;
  min-height: 150px;
  padding: 1rem;
  margin-bottom: 1rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-family: monospace;
  resize: vertical;
  box-sizing: border-box;
}

.parsed-output {
  width: 100%;
  overflow-x: auto;
  padding: 1rem;
  border: 1px solid #eee;
  border-radius: 4px;
  background-color: #f9f9f9;
  box-sizing: border-box;
}

.annotation-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1rem;
  font-size: 0.9rem;
  table-layout: fixed;
}

.annotation-table th,
.annotation-table td {
  padding: 0.5rem;
  text-align: left;
  border: 1px solid #ddd;
  overflow: hidden;
  text-overflow: ellipsis;
  word-wrap: break-word;
}

/* Column widths */
.annotation-table th:nth-child(1), /* ID */
.annotation-table td:nth-child(1) {
  width: 10%;
}

.annotation-table th:nth-child(2), /* Type */
.annotation-table td:nth-child(2) {
  width: 15%;
}

.annotation-table th:nth-child(3), /* Start */
.annotation-table td:nth-child(3),
.annotation-table th:nth-child(4), /* Stop */
.annotation-table td:nth-child(4) {
  width: 10%;
}

.annotation-table th:last-child,
.annotation-table td:last-child {
  color: #666;
  font-style: italic;
}

.length-mismatch {
  background-color: #ffebee;
  color: #c62828;
  font-weight: bold;
}

.mismatch-info {
  margin-left: 0.5rem;
  cursor: help;
}

/* Add transition for smoother color display */
.annotation-table td {
  transition: background-color 0.2s ease;
}
</style>

