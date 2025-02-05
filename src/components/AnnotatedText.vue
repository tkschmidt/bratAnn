<template>
  <div class="annotated-text-container">
    <div class="controls">
      <label for="offset">Position Offset:</label>
      <input 
        type="range" 
        id="offset" 
        v-model="offsetValue" 
        min="-100" 
        max="100" 
        step="1"
        class="offset-slider"
      />
      <span class="offset-value">{{ offsetValue }}</span>
    </div>
    <textarea 
      v-model="sourceText"
      @input="handleSourceTextInput"
      class="text-input"
      placeholder="Paste your source text here..."
    ></textarea>
    <div class="text-display-wrapper">
      <div class="line-numbers">
        <div v-for="(position, index) in lineStartPositions" 
             :key="index" 
             class="line-number">
          {{ position }}
        </div>
      </div>
      <div class="text-display">
        <template v-for="(chunk, index) in textChunks" :key="index">
          <span 
            v-if="chunk.annotation"
            class="highlighted-text annotation"
            :style="getAnnotationStyle(chunk.annotation.key)"
            :title="`${chunk.annotation.key}: ${chunk.annotation.value}`"
          >
            {{ chunk.text }}
          </span>
          <span v-else>{{ chunk.text }}</span>
        </template>
      </div>
    </div>

    <div class="string-matches">
      <h4>String Position Analysis</h4>
      
      <div class="fuzzy-controls">
        <div class="control-group">
          <label>
            Fuzzy Threshold:
            <input 
              type="range" 
              v-model.number="fuseThreshold" 
              min="0" 
              max="1" 
              step="0.01"
              class="threshold-slider"
            />
            <span class="threshold-value">{{ fuseThreshold.toFixed(2) }}</span>
          </label>
          <span class="control-hint">Lower values = stricter matching</span>
        </div>
        <div class="control-group">
          <label>
            Distance:
            <input 
              type="range" 
              v-model.number="fuseDistance" 
              min="0" 
              max="100" 
              step="1"
              class="threshold-slider"
            />
            <span class="threshold-value">{{ fuseDistance }}</span>
          </label>
          <span class="control-hint">Maximum edit distance allowed</span>
        </div>
      </div>
      <div class="column-toggles">
        <label v-for="(label, col) in columnLabels" :key="col" class="toggle-label">
          <input 
            type="checkbox" 
            v-model="visibleColumns[col]"
            :disabled="col === 'value'"  <!-- Always show value column -->
          >
          {{ label }}
        </label>
      </div>
      <div class="table-wrapper">
        <table class="matches-table" v-if="filteredMatches.length || sourceText">
          <thead>
            <tr>
              <th v-if="visibleColumns.id">ID</th>
              <th v-if="visibleColumns.type">Type</th>
              <th v-if="visibleColumns.value">Value</th>
              <th v-if="visibleColumns.annotatedPosition">Annotated Position</th>
              <th v-if="visibleColumns.foundPositions">Found Positions</th>
              <th v-if="visibleColumns.fuzzyMatches">Fuzzy Matches</th>
              <th v-if="visibleColumns.meta">Meta</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="match in filteredMatches" 
                :key="match.id"
                :class="{ 'position-mismatch': hasNoMatches(match) }">
              <td v-if="visibleColumns.id">{{ match.id }}</td>
              <td v-if="visibleColumns.type">{{ match.key }}</td>
              <td v-if="visibleColumns.value">{{ match.value }}</td>
              <td v-if="visibleColumns.annotatedPosition">{{ match.annotatedPosition }}</td>
              <td v-if="visibleColumns.foundPositions">
                <span v-if="match.exactPositions.length === 0">Not found</span>
                <span v-else>
                  {{ match.exactPositions.join(', ') }}
                  <span v-if="!match.exactPositions.includes(match.start_position)" 
                        class="mismatch-warning" 
                        :title="match.mismatchInfo">⚠️</span>
                </span>
              </td>
              <td v-if="visibleColumns.fuzzyMatches">
                <div v-for="(fuzzyMatch, index) in match.fuzzyMatches" 
                     :key="index" 
                     class="fuzzy-match">
                  <div class="match-score">
                    <span class="score-indicator" 
                          :class="getScoreClass(fuzzyMatch.score)"></span>
                    Score: {{ fuzzyMatch.score.toFixed(2) }}
                  </div>
                  <div class="context-entry">
                    <strong>@{{ fuzzyMatch.position }}:</strong> 
                    <span class="context-text" v-html="fuzzyMatch.formattedText"></span>
                  </div>
                </div>
              </td>
              <td v-if="visibleColumns.meta">{{ match.meta || '-' }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import Fuse from 'fuse.js'
import chroma from 'chroma-js'

const props = defineProps({
  annotations: {
    type: Array,
    default: () => []
  },
  defaultText: {
    type: String,
    default: ''
  }
})

const sourceText = ref(props.defaultText)
const offsetValue = ref(0)
const fuseThreshold = ref(0.1)
const fuseDistance = ref(1)
const idFilter = ref('')
const visibleColumns = ref({
  id: true,
  type: true,
  value: true,
  annotatedPosition: true,
  foundPositions: true,
  fuzzyMatches: true,
  meta: true
})

// Force recomputation when fuzzy parameters change
const fuzzyParams = computed(() => ({
  threshold: fuseThreshold.value,
  distance: fuseDistance.value
}))

// Calculate starting character position for each line
const lineStartPositions = computed(() => {
  if (!sourceText.value) return []
  
  const positions = []
  let currentPosition = 0
  const lines = sourceText.value.split('\n')
  
  lines.forEach(line => {
    positions.push(currentPosition)
    currentPosition += line.length + 1 // +1 for the newline character
  })
  
  return positions
})

// Get unique keys from annotations
const uniqueKeys = computed(() => {
  return [...new Set(props.annotations.map(ann => ann.key))].sort()  // Sort for consistent ordering
})

// Generate color map using chroma-js
const colorMap = computed(() => {
  const keys = uniqueKeys.value
  if (keys.length === 0) return new Map()

  // Use a qualitative color scale that's better for categorical data
  let colors
  if (keys.length <= 8) {
    colors = chroma.brewer.Set2  // For fewer categories
  } else if (keys.length <= 12) {
    colors = chroma.brewer.Set3  // For medium number of categories
  } else {
    // For many categories, generate custom rainbow scale
    colors = chroma
      .scale(['#00429d', '#93003a', '#35b779', '#ff8e6d', '#ffd700', '#93003a'])
      .mode('lch')
      .colors(keys.length)
  }
  
  // Create a map of key -> color
  return new Map(keys.map((key, index) => [
    key, 
    {
      backgroundColor: chroma(colors[index % colors.length]).alpha(0.2).css(),
      ':hover': {
        backgroundColor: chroma(colors[index % colors.length]).alpha(0.4).css()
      }
    }
  ]))
})

// Get styles for a specific annotation type
const getAnnotationStyle = (type) => {
  return colorMap.value.get(type) || {
    backgroundColor: 'rgba(200, 200, 200, 0.2)',
    ':hover': {
      backgroundColor: 'rgba(200, 200, 200, 0.4)'
    }
  }
}

const adjustedAnnotations = computed(() => {
  return props.annotations.map(ann => ({
    ...ann,
    start_position: ann.start_position + parseInt(offsetValue.value),
    stop_position: ann.stop_position + parseInt(offsetValue.value)
  }))
})

const textChunks = computed(() => {
  if (!sourceText.value) return []
  
  // Create sorted positions array using adjusted annotations
  const positions = []
  adjustedAnnotations.value.forEach(ann => {
    // Ensure positions don't go below 0 or beyond text length
    const start = Math.max(0, Math.min(ann.start_position, sourceText.value.length))
    const stop = Math.max(0, Math.min(ann.stop_position, sourceText.value.length))
    
    positions.push({
      position: start,
      type: 'start',
      annotation: ann
    })
    positions.push({
      position: stop,
      type: 'end',
      annotation: ann
    })
  })
  
  positions.sort((a, b) => a.position - b.position)
  
  // Split text into chunks
  const chunks = []
  let currentPos = 0
  let currentAnnotation = null
  
  positions.forEach(pos => {
    // Add text chunk before this position
    if (pos.position > currentPos) {
      chunks.push({
        text: sourceText.value.substring(currentPos, pos.position),
        annotation: currentAnnotation
      })
    }
    
    // Update current annotation
    if (pos.type === 'start') {
      currentAnnotation = pos.annotation
    } else {
      currentAnnotation = null
    }
    
    currentPos = pos.position
  })
  
  // Add remaining text
  if (currentPos < sourceText.value.length) {
    chunks.push({
      text: sourceText.value.substring(currentPos),
      annotation: null
    })
  }
  
  return chunks
})

const handleSourceTextInput = (event) => {
  sourceText.value = normalizeText(event.target.value)
}

const findFuzzyMatches = (text, searchValue) => {
  // Text is already normalized when stored
  const possibleMatches = []
  const windowSize = searchValue.length * 2
  
  for (let i = 0; i < text.length - searchValue.length + 1; i++) {
    possibleMatches.push({
      text: text.slice(i, i + windowSize),
      position: i
    })
  }

  const fuse = new Fuse(possibleMatches, {
    keys: ['text'],
    includeScore: true,
    threshold: fuzzyParams.value.threshold,
    distance: fuzzyParams.value.distance,
    location: 0
  })

  const results = fuse.search(searchValue)
  
  return results.map(result => {
    const match = result.item
    const score = 1 - result.score
    
    const matchText = match.text.slice(0, searchValue.length)
    const contextBefore = text.slice(Math.max(0, match.position - 20), match.position)
    const contextAfter = text.slice(match.position + searchValue.length, match.position + searchValue.length + 20)
    
    return {
      position: match.position,
      score,
      formattedText: `${contextBefore}<mark>${matchText}</mark>${contextAfter}`,
      context: match.text
    }
  })
  .filter(match => match.score > 0.3)
  .sort((a, b) => b.score - a.score)
}

const findExactOccurrences = (str, searchValue) => {
  const positions = []
  let pos = 0
  
  const normalizedText = str.replace(/\r\n/g, '\n')
  const normalizedSearch = searchValue.replace(/\r\n/g, '\n')
  
  while (true) {
    pos = normalizedText.indexOf(normalizedSearch, pos)
    if (pos === -1) break
    positions.push(pos)
    pos += 1
  }
  return positions
}

const stringMatches = computed(() => {
  if (!sourceText.value) return []
  
  // Include fuzzyParams in the computation to force reactivity
  const params = fuzzyParams.value
  
  return props.annotations.map(ann => {
    const exactPositions = findExactOccurrences(sourceText.value, ann.value)
    const fuzzyMatches = findFuzzyMatches(sourceText.value, ann.value)
    const annotatedPosition = `${ann.start_position}-${ann.stop_position}`
    
    return {
      id: ann.id || '',
      key: ann.key || '',
      value: ann.value || '',
      annotatedPosition,
      start_position: ann.start_position,
      exactPositions,
      fuzzyMatches,
      hasPositionMismatch: !exactPositions.includes(ann.start_position),
      mismatchInfo: !exactPositions.includes(ann.start_position)
        ? `Annotated position (${ann.start_position}) not found in exact matches`
        : '',
      meta: ann.meta
    }
  })
})

const getScoreClass = (score) => {
  if (score >= 0.9) return 'score-high'
  if (score >= 0.5) return 'score-medium'
  return 'score-low'
}

const hasNoMatches = (match) => {
  return match.exactPositions.length === 0 && match.fuzzyMatches.length === 0
}

const filteredMatches = computed(() => {
  if (!stringMatches.value) return []
  
  return stringMatches.value.filter(match => {
    if (!idFilter.value) return true
    return match.id.toLowerCase().includes(idFilter.value.toLowerCase())
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
</script>

<style scoped>
.annotated-text-container {
  width: 100%;
  max-width: 100%;
  padding: 1rem;
  box-sizing: border-box;
}

.controls {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.offset-slider {
  flex: 1;
  max-width: 300px;
  min-width: 100px;
}

.offset-value {
  min-width: 3em;
  text-align: right;
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

.text-display-wrapper {
  display: flex;
  gap: 1rem;
  border: 1px solid #eee;
  border-radius: 4px;
  background-color: white;
  overflow-x: auto;
  width: 100%;
  box-sizing: border-box;
}

.line-numbers {
  padding: 1rem 0.5rem;
  background-color: #f5f5f5;
  border-right: 1px solid #eee;
  font-family: monospace;
  font-size: 0.9em;
  color: #666;
  text-align: right;
  min-width: 4em;
  user-select: none;
  flex-shrink: 0;
}

.line-number {
  padding: 0 0.5rem;
  line-height: 1.6;
}

.text-display {
  flex: 1;
  padding: 1rem;
  line-height: 1.6;
  white-space: pre-wrap;
  font-family: monospace;
  overflow-x: hidden;
  word-wrap: break-word;
}

.highlighted-text {
  padding: 2px 0;
  border-radius: 2px;
  cursor: help;
}

.string-matches {
  margin-top: 2rem;
  padding: 1rem;
  border: 1px solid #eee;
  border-radius: 4px;
  width: 100%;
  box-sizing: border-box;
}

.fuzzy-controls {
  margin: 1rem 0;
  padding: 1rem;
  background-color: #f5f5f5;
  border-radius: 4px;
  width: 100%;
  box-sizing: border-box;
}

.control-group {
  margin: 0.75rem 0;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.control-hint {
  font-size: 0.8rem;
  color: #666;
  font-style: italic;
}

.threshold-slider {
  width: 200px;
  margin: 0 1rem;
  vertical-align: middle;
}

.threshold-value {
  min-width: 2.5em;
  display: inline-block;
  text-align: right;
}

.fuzzy-match {
  margin-bottom: 1rem;
  padding: 0.5rem;
  border-left: 3px solid #2196f3;
  background-color: #f8f9fa;
  max-width: 100%;
  overflow-wrap: break-word;
  word-break: break-all;
}

.match-score {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.8rem;
  color: #666;
  margin-bottom: 0.25rem;
}

.score-indicator {
  display: inline-block;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  border: 1px solid rgba(0, 0, 0, 0.1);
}

.score-high {
  background-color: #4caf50;  /* Green */
  box-shadow: 0 0 4px rgba(76, 175, 80, 0.4);
}

.score-medium {
  background-color: #ffc107;  /* Yellow */
  box-shadow: 0 0 4px rgba(255, 193, 7, 0.4);
}

.score-low {
  background-color: #f44336;  /* Red */
  box-shadow: 0 0 4px rgba(244, 67, 54, 0.4);
}

mark {
  background-color: #fff3cd;
  padding: 0.1rem 0.2rem;
  border-radius: 2px;
}

.context-entry {
  margin-bottom: 0.5rem;
  font-family: monospace;
  font-size: 0.85rem;
  white-space: pre-wrap;
  word-break: break-word;
}

.context-text {
  color: #666;
}

.column-toggles {
  margin-top: 1rem;
  margin-bottom: 1rem;
}

.toggle-label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.table-wrapper {
  position: relative;
}

.matches-table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
}

.matches-table thead {
  position: sticky;
  top: 0;
  z-index: 1;
  background: white;
}

.matches-table th {
  padding: 0.75rem 0.5rem;
  text-align: left;
  border-bottom: 2px solid #ddd;
  background: white;
  box-shadow: 0 2px 2px -1px rgba(0, 0, 0, 0.1);
}

.matches-table td {
  padding: 0.5rem;
  text-align: left;
  border-bottom: 1px solid #ddd;
  background: white;
}

.position-mismatch td {
  background-color: #fff3f3;
}

.mismatch-warning {
  margin-left: 0.5rem;
  cursor: help;
}

.annotation {
  border-radius: 2px;
  transition: background-color 0.2s ease;
}
</style> 