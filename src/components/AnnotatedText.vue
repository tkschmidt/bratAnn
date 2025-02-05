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
</style> 