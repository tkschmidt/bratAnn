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
            </tr>
          </thead>
          <tbody>
            <tr v-for="ann in parsedAnnotations" :key="ann.id">
              <td>{{ ann.id }}</td>
              <td>{{ ann.key }}</td>
              <td>{{ ann.start_position }}</td>
              <td>{{ ann.stop_position }}</td>
              <td>{{ ann.value }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <div class="text-visualization">
      <h3>Text Visualization</h3>
      <AnnotatedText :annotations="parsedAnnotations" />
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import AnnotatedText from './AnnotatedText.vue'

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

.annotation-input, .text-visualization {
  margin-bottom: 2rem;
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

.annotation-table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1rem;
  font-size: 0.9rem;
}

.annotation-table th,
.annotation-table td {
  padding: 0.5rem;
  text-align: left;
  border: 1px solid #ddd;
}

.annotation-table th {
  background-color: #f5f5f5;
  font-weight: bold;
}

.annotation-table tr:nth-child(even) {
  background-color: #fafafa;
}

.annotation-table tr:hover {
  background-color: #f0f0f0;
}
</style>

