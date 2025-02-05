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
      <table class="matches-table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Value</th>
            <th>Annotated Position</th>
            <th>Found Positions</th>
            <th>Context</th>
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
              <span v-if="match.positions.length === 0">Not found</span>
              <span v-else>
                {{ match.positions.join(', ') }}
                <span v-if="match.hasPositionMismatch" 
                      class="mismatch-warning" 
                      :title="match.mismatchInfo">⚠️</span>
              </span>
            </td>
            <td>
              <div v-for="(ctx, index) in match.positionsWithContext" 
                   :key="index" 
                   class="context-entry">
                <strong>@{{ ctx.position }}:</strong> 
                <span class="context-text">{{ ctx.context }}</span>
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

const props = defineProps({
  annotations: {
    type: Array,
    default: () => []
  }
})

const sourceText = ref('')
const offsetValue = ref(0)

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

const findAllOccurrences = (str, searchValue) => {
  const positions = []
  let pos = 0
  
  // Convert both strings to normalized form to handle potential line ending differences
  const normalizedText = str.replace(/\r\n/g, '\n')
  const normalizedSearch = searchValue.replace(/\r\n/g, '\n')
  
  while (true) {
    pos = normalizedText.indexOf(normalizedSearch, pos)
    if (pos === -1) break
    
    // Also try matching with different line endings
    const alternateMatch = str.slice(pos, pos + searchValue.length)
    if (alternateMatch.replace(/\r\n/g, '\n') === normalizedSearch) {
      positions.push(pos)
    }
    
    pos += 1 // Move to next character to continue search
  }
  return positions
}

const stringMatches = computed(() => {
  if (!sourceText.value) return []
  
  return props.annotations.map(ann => {
    // Get all positions where the value appears
    const positions = findAllOccurrences(sourceText.value, ann.value)
    const annotatedPosition = `${ann.start_position}-${ann.stop_position}`
    
    // Check if the annotated position matches any found position
    const hasPositionMismatch = !positions.includes(ann.start_position)
    
    // Create context snippets for each position
    const positionsWithContext = positions.map(pos => {
      const start = Math.max(0, pos - 20)
      const end = Math.min(sourceText.value.length, pos + ann.value.length + 20)
      const context = sourceText.value.slice(start, end)
        .replace(/\n/g, '↵') // Show line breaks with symbol
        .replace(/\r/g, '') // Remove carriage returns from display
      
      return {
        position: pos,
        context: `...${context}...`
      }
    })
    
    return {
      id: ann.id,
      value: ann.value.replace(/\n/g, '↵'), // Show line breaks in value
      annotatedPosition,
      positions,
      positionsWithContext,
      hasPositionMismatch,
      mismatchInfo: hasPositionMismatch 
        ? `Annotated position (${ann.start_position}) not found in actual string positions`
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

.context-entry {
  margin-bottom: 0.5rem;
  font-family: monospace;
  font-size: 0.85rem;
}

.context-text {
  color: #666;
}

.context-entry strong {
  color: #000;
}
</style> 