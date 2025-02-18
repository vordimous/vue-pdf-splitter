<script setup>
import { ref } from 'vue'
import PdfSplitter from './components/PdfSplitter.vue'
const scale = ref(.2)
const urls = [
  'https://pdf-lib.js.org/assets/american_flag.pdf',
  'https://pdf-lib.js.org/assets/us_constitution.pdf',
  'https://mozilla.github.io/pdf.js/web/compressed.tracemonkey-pldi-09.pdf', 
  // 'https://pdf-lib.js.org/assets/with_large_page_count.pdf', // Uncomment to test performance with a large PDF
]

</script>

<template>
  <main>
    <input type="range" min=".15" max="1" step=".05" v-model="scale" class="slider">
    <PdfSplitter :urls="urls" :scale="scale" />
  </main>
</template>
