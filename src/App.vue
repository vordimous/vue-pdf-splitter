<script setup>
import PdfSplitter from './components/PdfSplitter.vue'
const urls = [
  'https://pdf-lib.js.org/assets/american_flag.pdf',
  'https://pdf-lib.js.org/assets/us_constitution.pdf',
  'https://mozilla.github.io/pdf.js/web/compressed.tracemonkey-pldi-09.pdf', 
  // 'https://pdf-lib.js.org/assets/with_large_page_count.pdf',
]
</script>

<template>

  <main>
    <PdfSplitter :preLoadPdfUrls="urls" />
  </main>
</template>

<style scoped>
header {
  line-height: 1.5;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }
}
</style>
