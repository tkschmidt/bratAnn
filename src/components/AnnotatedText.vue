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
      placeholder="Paste your source text here..."
      class="text-input"
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
            class="highlighted-text"
            :style="{ backgroundColor: getColorForKey(chunk.annotation.key) }"
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
        <label>
          Fuzzy Threshold:
          <input 
            type="range" 
            v-model="fuseThreshold" 
            min="0" 
            max="1" 
            step="0.1"
            class="threshold-slider"
          />
          <span class="threshold-value">{{ fuseThreshold }}</span>
        </label>
      </div>
      <table class="matches-table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Value</th>
            <th>Annotated Position</th>
            <th>Exact Matches</th>
            <th>Fuzzy Matches</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="match in stringMatches" 
              :key="match.id"
              :class="{ 'position-mismatch': match.hasPositionMismatch }">
            <td>{{ match.id }}</td>
            <td>{{ match.value }}</td>
            <td>{{ match.annotatedPosition }}</td>
            <td>
              <span v-if="match.exactPositions.length === 0">Not found</span>
              <span v-else>
                {{ match.exactPositions.join(', ') }}
                <span v-if="match.hasPositionMismatch" 
                      class="mismatch-warning" 
                      :title="match.mismatchInfo">⚠️</span>
              </span>
            </td>
            <td>
              <div v-for="(fuzzyMatch, index) in match.fuzzyMatches" 
                   :key="index" 
                   class="fuzzy-match">
                <div class="match-score">
                  Score: {{ fuzzyMatch.score.toFixed(2) }}
                </div>
                <div class="context-entry">
                  <strong>@{{ fuzzyMatch.position }}:</strong> 
                  <span class="context-text" v-html="fuzzyMatch.formattedText"></span>
                </div>
              </div>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import Fuse from 'fuse.js'

const props = defineProps({
  annotations: {
    type: Array,
    default: () => []
  }
})

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

const sourceText = ref(defaultText)
const offsetValue = ref(0)
const fuseThreshold = ref(0.3)

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

// Create a simple color hash for different annotation keys
const getColorForKey = (key) => {
  const colors = {
    'Compound': '#ffcdd2',
    'ConcentrationOfCompound': '#c8e6c9',
    'Time': '#bbdefb',
    'Temperature': '#fff9c4',
    'SpikedCompound': '#e1bee7',
    'NumberOfSamples': '#b2dfdb',
    'Label': '#ffccbc'
  }
  return colors[key] || '#f5f5f5'
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

const findFuzzyMatches = (text, searchValue) => {
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
    threshold: fuseThreshold.value,
    location: 0,
    distance: Math.floor(searchValue.length / 3), // Reduced distance
    minMatchCharLength: Math.floor(searchValue.length * 0.8), // Minimum length of match
    shouldSort: true,
    findAllMatches: false
  })

  const results = fuse.search(searchValue)
  
  // Filter out overlapping matches
  const filteredResults = results.reduce((acc, result) => {
    const match = result.item
    const score = 1 - result.score
    
    // Skip if score is too low
    if (score < 0.6) return acc
    
    // Check if this match overlaps with any existing matches
    const hasOverlap = acc.some(existing => {
      const existingStart = existing.position
      const existingEnd = existingStart + searchValue.length
      const currentStart = match.position
      const currentEnd = currentStart + searchValue.length
      
      // Check for any overlap
      return !(currentEnd < existingStart || currentStart > existingEnd) &&
             // Allow small distance for different matches
             Math.abs(currentStart - existingStart) < searchValue.length / 2
    })
    
    if (!hasOverlap) {
      const matchText = match.text.slice(0, searchValue.length)
      const contextBefore = text.slice(Math.max(0, match.position - 20), match.position)
      const contextAfter = text.slice(match.position + searchValue.length, match.position + searchValue.length + 20)
      
      acc.push({
        position: match.position,
        score,
        formattedText: `${contextBefore}<mark>${matchText}</mark>${contextAfter}`,
        context: match.text
      })
    }
    
    return acc
  }, [])
  
  return filteredResults
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
  
  return props.annotations.map(ann => {
    const exactPositions = findExactOccurrences(sourceText.value, ann.value)
    const fuzzyMatches = findFuzzyMatches(sourceText.value, ann.value)
    const annotatedPosition = `${ann.start_position}-${ann.stop_position}`
    const hasPositionMismatch = !exactPositions.includes(ann.start_position)
    
    return {
      id: ann.id,
      value: ann.value.replace(/\n/g, '↵'),
      annotatedPosition,
      exactPositions,
      fuzzyMatches,
      hasPositionMismatch,
      mismatchInfo: hasPositionMismatch 
        ? `Annotated position (${ann.start_position}) not found in exact matches`
        : ''
    }
  })
})
</script>

<style scoped>
.annotated-text-container {
  width: 100%;
  padding: 1rem;
}

.controls {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1rem;
}

.offset-slider {
  flex: 1;
  max-width: 300px;
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
}

.text-display-wrapper {
  display: flex;
  gap: 1rem;
  border: 1px solid #eee;
  border-radius: 4px;
  background-color: white;
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
}

.matches-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1rem;
  font-size: 0.9rem;
}

.matches-table th,
.matches-table td {
  padding: 0.5rem;
  text-align: left;
  border: 1px solid #ddd;
  vertical-align: top;
}

.matches-table th {
  background-color: #f5f5f5;
  font-weight: bold;
}

.position-mismatch td {
  background-color: #fff3f3;
}

.mismatch-warning {
  margin-left: 0.5rem;
  cursor: help;
}

.fuzzy-controls {
  margin: 1rem 0;
  padding: 0.5rem;
  background-color: #f5f5f5;
  border-radius: 4px;
}

.threshold-slider {
  width: 200px;
  margin: 0 1rem;
}

.fuzzy-match {
  margin-bottom: 1rem;
  padding: 0.5rem;
  border-left: 3px solid #2196f3;
  background-color: #f8f9fa;
}

.match-score {
  font-size: 0.8rem;
  color: #666;
  margin-bottom: 0.25rem;
}

mark {
  background-color: #fff3cd;
  padding: 0.1rem 0.2rem;
  border-radius: 2px;
}

.context-entry {
  font-family: monospace;
  font-size: 0.85rem;
  white-space: pre-wrap;
}
</style> 