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
              <th>Length</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="ann in annotationsWithLength" :key="ann.id">
              <td>{{ ann.id }}</td>
              <td>{{ ann.key }}</td>
              <td>{{ ann.start_position }}</td>
              <td>{{ ann.stop_position }}</td>
              <td>{{ ann.value }}</td>
              <td :class="{ 'length-mismatch': ann.hasLengthMismatch }">
                {{ ann.valueLength }}
                <span v-if="ann.hasLengthMismatch" class="mismatch-info" :title="ann.mismatchInfo">
                  ⚠️ (Δ{{ ann.lengthDelta }})
                </span>
              </td>
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

const defaultAnnotations = `T1	Compound 67 71	DMSO
T2	ConcentrationOfCompound 94 106	10 pmol/µl
T3	Time 132 140	30 min
T4	Temperature 144 159	room temperature
T5	Compound 179 187	formic acid
T6	ConcentrationOfCompound 208 219	1 pmol/µl
T7	Temperature 238 243	-20 °C
T8	ConcentrationOfCompound 249 252	10 µl
T9	SpikedCompound 308 328	retention time (RT) standards
T10	SpikedCompound 366 394	RT peptides (JPT Peptide Technologies)
T11	NumberOfSamples 407 409	66
T12	SpikedCompound 529 539	Retention time
T13	NumberOfSamples 550 552	15
T14	Label 553 566	13C-labeled
T15	ConcentrationOfCompound 571 579	100 fmol
T16	ConcentrationOfCompound 477 485	200 fmol
T17	NumberOfSamples 580 583	per peptide
T18	Temperature 643 648	-20 °C
T19	NumberOfSamples 275 283	96-well plate
T20	NumberOfSamples 700 708	96-well plates`

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
}

const handleTextChange = () => {
  const lines = inputText.value.split('\n').filter(line => line.trim())
  
  parsedAnnotations.value = lines.map(line => {
    const [id, keyWithPosition, ...valueParts] = line.split('\t')
    const [key, startPos, endPos] = keyWithPosition.split(' ')
    const value = normalizeText(valueParts.join('\t').trim())
    
    return {
      id,
      key,
      start_position: parseInt(startPos),
      stop_position: parseInt(endPos),
      value
    }
  })
}

// Initialize with default annotations
handleTextChange()
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

.length-mismatch {
  background-color: #ffebee;
  color: #c62828;
  font-weight: bold;
}

.mismatch-info {
  margin-left: 0.5rem;
  cursor: help;
}
</style>

