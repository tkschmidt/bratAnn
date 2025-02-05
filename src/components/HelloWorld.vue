<script setup>
import BratAnn from './bratAnn.vue'
</script>

<template>
  <div class="brat-container">
    <h2>BRAT Annotation Checker</h2>
    <p class="tool-description">
      A tool for checking and validating BRAT annotations. Paste your source text and annotations to verify positions, 
      find fuzzy matches, and detect potential misalignments. It can be used with the 
      <a href="https://chatgpt.com/g/g-67a2271be1cc81919b0cdef27ff7b952-proteomics-sdrf-ner-extractor" 
         target="_blank" 
         rel="noopener noreferrer">
        Proteomics SDRF NER Extractor GPT
      </a>
    </p>
    <BratAnn />
  </div>
</template>

<style scoped>
.brat-container {
  margin: 2rem 0;
  padding: 1rem;
  border: 1px solid #ddd;
  border-radius: 8px;
}

.tool-description {
  color: #666;
  margin: 1rem 0 2rem;
  line-height: 1.5;
  max-width: fit-content;
}
</style>
