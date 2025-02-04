<template>
    <div class="pdf-document" v-for="(doc, index) in docs" :id="`new-pdf-document-${index}`" :key="index"
        :ref="(el) => { initUseDropZone(doc, el); initUseSortable(doc, el) }":class="{ 'pdf-document-drop': doc.hover }">
      <div v-if="doc.pages.length === 0" class="pdf-document-new">
        <div>Drag Here to Create</div>
      </div>
      <div v-for="page in doc.pages" :key="page" class="pdf-document-page">
        <VuePDF :pdf="pdf" :page="page" :scale="scale" />
      </div>
    </div>
</template>

<script lang="ts" setup>
import { nextTick, watch, reactive, Reactive, ref, Ref } from 'vue'
import { VuePDF, usePDF } from '@tato30/vue-pdf'

const { pdf, pages: totalPdfPages } = usePDF('https://mozilla.github.io/pdf.js/web/compressed.tracemonkey-pldi-09.pdf')
import { useDropZone } from '@vueuse/core'
import { useSortable, moveArrayElement } from '@vueuse/integrations/useSortable'

type Doc = {
  index: number;
  pages: Reactive<number[]>;
  name: string;
  hover: boolean;
};
function addEmptyDoc() {
  docs.push({ index: docs.length, pages: [], name: "", hover: false, })
}
const docs: Reactive<Doc[]> = reactive<Reactive<Doc[]>>([])
addEmptyDoc()

// when the PDF has updated the pages, load them into the splitter
watch(totalPdfPages, (p) => {
  docs[0].pages.push(...Array.from({ length: p - 1 }, (_, index) => index + 1))
  docs[0].name = "tracemonkey-pldi-09.pdf"
  nextTick(() => {
    addEmptyDoc()
  })
})


var dragOrig: Doc | undefined
var dragDest: Doc | undefined

function initUseDropZone(doc: Doc, el: HTMLElement | any | null) {
  useDropZone(el, {
    onDrop: () => {
      dragDest = doc
      doc.hover = false
    },
    onOver: () => {
      nextTick(() => {
        doc.hover = true
      })
    },
    onLeave: () => {
      nextTick(() => {
        doc.hover = false
      })
    }
  })
}
function initUseSortable(doc: Doc, el: HTMLElement | any | null) {
  useSortable(el, doc.pages, {
    onStart: () => {
      dragOrig = doc
      dragDest = doc
    },
    onUpdate: (e) => {
      // only update the sortable order if the drop is in the same document
      if (dragOrig !== undefined && dragDest !== undefined && dragOrig === dragDest
        && e.oldIndex !== undefined && e.newIndex !== undefined) {
        moveArrayElement(doc.pages, e.oldIndex, e.newIndex)
      }
    },
    onEnd: (e) => {
      if (dragOrig !== undefined && dragDest !== undefined
        && dragOrig !== dragDest && e.oldIndex !== undefined) {
        // move the page from its starting index in the Origin to the end of the Dest
        dragDest.pages.push(dragOrig.pages[e.oldIndex])
        dragOrig.pages.splice(e.oldIndex, 1);
      }
      // Add new doc
      var emptyDocs = docs.filter((a) => a.pages.length === 0).length
      if (emptyDocs === 0) {
        addEmptyDoc()
      }
    },
  })
}

const scale = ref(.15)

</script>

<style>
body {
    background-color: whitesmoke;
}
.pdf-document {
    background-color: lightgray;
    display: flex;
    flex-wrap: wrap;
    justify-content: left;
    align-items: center;
    width: 80vw;
    min-height: 75px;
    margin: 5px;
}

.pdf-document .pdf-document-new {
  border: 1px dashed grey;
  color: black;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100px;
}

.pdf-document .pdf-document-page {
  cursor: grab;
  background-color: grey;
  padding: 3px;
  margin: 3px;
}
.pdf-document-drop {
  background-color: var(--gray-200);
}
</style>
