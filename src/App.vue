<script setup>
import PdfSplitter from './components/PdfSplitter.vue'
const urls = [
  'https://pdf-lib.js.org/assets/american_flag.pdf',
  'https://pdf-lib.js.org/assets/us_constitution.pdf',
  'https://mozilla.github.io/pdf.js/web/compressed.tracemonkey-pldi-09.pdf', 
  // 'https://pdf-lib.js.org/assets/with_large_page_count.pdf', // Uncomment to test performance with a large PDF
]
</script>

<template>
  <main>
    <PdfSplitter :preLoadPdfUrls="urls" />
  </main>
</template>
