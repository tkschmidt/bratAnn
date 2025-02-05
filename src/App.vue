<script setup>
import BratChecker from './components/BratChecker.vue'
</script>

<template>
  <div class="app-container">
    <BratChecker />
  </div>
</template>

<style>
/* Reset and base styles */
:root {
  color-scheme: light;
}

body {
  margin: 0;
  padding: 0;
  background-color: #ffffff;
  color: #000000;
}

.app-container {
  width: 100%;
  min-height: 100vh;
  background-color: #ffffff;
  color: #000000;
}

/* Ensure all inputs and textareas have consistent colors */
input, textarea {
  background-color: #ffffff;
  color: #000000;
  border-color: #cccccc;
}

/* Ensure text is visible in highlighted sections */
.highlighted-text {
  color: #000000;
}
</style>
