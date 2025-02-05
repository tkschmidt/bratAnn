<template>
  <div class="brat-annotation">
    <textarea 
      v-model="inputText" 
      placeholder="Paste your BRAT annotations here..."
      class="text-input"
      @input="handleTextChange"
    ></textarea>
    <div class="parsed-output">
      <p>Parsed Annotations:</p>
      <pre>{{ JSON.stringify(parsedAnnotations, null, 2) }}</pre>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const inputText = ref('')
const parsedAnnotations = ref([])

const handleTextChange = () => {
  const lines = inputText.value.split('\n').filter(line => line.trim())
  
  parsedAnnotations.value = lines.map(line => {
    const [id, keyWithPosition, ...valueParts] = line.split('\t')
    
    // Split the key and position part (e.g., "Compound 67 71")
    const [key, startPos, endPos] = keyWithPosition.split(' ')
    
    // Combine remaining parts as the value
    const value = valueParts.join('\t').trim()
    
    return {
      id,
      key,
      start_position: parseInt(startPos),
      stop_position: parseInt(endPos),
      value
    }
  })
}
</script>

<style scoped>
.brat-annotation {
  width: 100%;
  padding: 1rem;
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

.parsed-output {
  padding: 1rem;
  border: 1px solid #eee;
  border-radius: 4px;
  background-color: #f9f9f9;
  overflow-x: auto;
}

pre {
  white-space: pre-wrap;
  word-wrap: break-word;
  font-family: monospace;
}
</style>

