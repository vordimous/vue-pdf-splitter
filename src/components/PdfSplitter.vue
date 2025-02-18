<template>
  <div
    v-for="(pageGroup, pgIndex) in allPageGroups"
    :id="`new-pdf-document-${pgIndex}`" 
    :key="pgIndex"
    class="pdf-document" 
    :class="{ 'pdf-document-drop': pageGroup.hover }">
    <div class="pdf-document-header">
      <button @click="() => downloadDoc(pageGroup)">Download</button>
      <input type="text" style="width: 100%;" :value="pageGroup.name" />
      <button @click="() => removeDoc(pgIndex)">Delete</button>
    </div>
    <div :ref="(el: any) => pageGroup.pageElRef = el" class="pdf-document-body">
      <div v-if="pageGroup.pages.length === 0" class="pdf-document-empty">
        <p>Drag page here to add to New Doc</p>
      </div>
      <div  v-for="(page, pIndex) in pageGroup.pages" :key="page.pageNumber" class="pdf-document-page-group">
        <div class="pdf-document-page">
          <VuePDF :pdf="pdfPageViewer" :page="page.pageNumber" :scale="scale" :rotation="page.rotation" class="pdf-viewer" />
          <div class="page-buttons">
            <button @click="() => page.rotation -= 90"><</button><button @click="() => page.rotation += 90">></button><button
            @click="removePage(pageGroup, pIndex)">x</button>
          </div>
        </div>
        <div v-if="pIndex != pageGroup.pages.length - 1" class="pdf-document-split" @click="splitDoc(pageGroup, pIndex)">
          <div class="dashed-line"></div>
          <div class="dashed-spacer"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { GlobalWorkerOptions, PDFDocumentLoadingTask, getDocument } from 'pdfjs-dist';
import LegacyWorker from 'pdfjs-dist/legacy/build/pdf.worker.min?url';
GlobalWorkerOptions.workerSrc = LegacyWorker
import { PDFDocument, degrees } from 'pdf-lib'
import { watch, nextTick, Reactive, ref, Ref, onMounted } from 'vue'
import { VuePDF } from '@tato30/vue-pdf'
import { useDropZone } from '@vueuse/core'
import { useSortable, moveArrayElement } from '@vueuse/integrations/useSortable'

const { urls, scale } = defineProps({
  urls: {
    type: Array<string>,
    default: () => [],
  },
  scale: {
    type: Number,
    default: () => .2,
  },
})

const pdfPageViewer: Ref<PDFDocumentLoadingTask | undefined> = ref()
var pdfDataDocument: PDFDocument

async function addPdfPagesFromUrl(url: string): Promise<number> {
  const arrayBuffer = await fetch(url).then(res => res.arrayBuffer())
  return await addPdfPages(arrayBuffer)
}

async function addPdfPages(arrayBuffer: ArrayBuffer): Promise<number> {
  var pdfDocument = await PDFDocument.load(arrayBuffer)
  var copiedPages = await pdfDataDocument.copyPages(pdfDocument, numArray(pdfDocument.getPageCount()))
  copiedPages.forEach((page) => {
    pdfDataDocument.addPage(page)
  })
  return copiedPages.length
}

async function renderPdfPages() {
  const loadingTask = getDocument(await pdfDataDocument.save())
  pdfPageViewer.value = loadingTask
}

function numArray(size: number, shift: number = 0): number[] {
  return Array.from({ length: size }, (_, index) => index + shift)
}

type Page = {
  pageNumber: number;
  rotation: number;
};
type PageGroup = {
  index: number;
  pages: Reactive<Page[]>;
  name: string;
  hover: boolean;
  pageElRef: HTMLElement | any;
};

function addPageGroup(pg: PageGroup | any) {
  allPageGroups.value.push({
    pages: [],
    name: "",
    ...pg,
    hover: false,
  })
  nextTick(() => {
    allPageGroups.value.forEach(initDragDrop);
  })
}

const allPageGroups: Ref<PageGroup[]> = ref<PageGroup[]>([])

onMounted(async () => {
  pdfDataDocument = await PDFDocument.create()
  downloadPdfs(urls)
})

watch(() => urls, async (newUrls) => await downloadPdfs(newUrls), { deep: true })
const pageScale = ref(scale || .20)
watch(() => scale, (newScale) => {pageScale.value = newScale || pageScale.value} )

async function downloadPdfs(urls: string[] | undefined) {
  var loadedDocs: { url: string, numOfPages: number }[] = []
  await Promise.all((urls || []).map(async (url) => {
    try {
      var numOfPages = await addPdfPagesFromUrl(url)
      loadedDocs.push({
        url,
        numOfPages
      })
    } catch (e) {
      console.error(e)
    }
  }))

  await renderPdfPages()
  var allPageNums = numArray(pdfDataDocument.getPageCount(), 1)
  loadedDocs.forEach(({
    url,
    numOfPages
  }) => {

    addPageGroup({
      pages: numArray(numOfPages, 1).map(() => ({ pageNumber: allPageNums.shift(), rotation: 0 })),
      name: url.substring(url.lastIndexOf('/') + 1),
    })
  })
}

var dragOrig: PageGroup | undefined
var dragDest: PageGroup | undefined

function initDragDrop(pg: PageGroup) {
  if (pg.pageElRef) {
    initUseDropZone(pg)
    initUseSortable(pg)
  }
}
function initUseDropZone(pageGroup: PageGroup) {
  useDropZone(pageGroup.pageElRef, {
    onDrop: () => {
      dragDest = pageGroup
      pageGroup.hover = false
    },
    onOver: () => {
      nextTick(() => {
        pageGroup.hover = true
      })
    },
    onLeave: () => {
      nextTick(() => {
        pageGroup.hover = false
      })
    }
  })
}
function initUseSortable(pageGroup: PageGroup) {
  useSortable(pageGroup.pageElRef, pageGroup.pages, {
    onStart: () => {
      dragOrig = pageGroup
      dragDest = pageGroup
    },
    onUpdate: (e) => {
      // only update the sortable order if the drop is in the same document
      if (dragOrig !== undefined && dragDest !== undefined && dragOrig === dragDest
        && e.oldIndex !== undefined && e.newIndex !== undefined) {
        moveArrayElement(pageGroup.pages, e.oldIndex, e.newIndex)
      }
    },
    onEnd: (e) => {
      if (dragOrig !== undefined && dragDest !== undefined
        && dragOrig !== dragDest && e.oldIndex !== undefined) {
        // move the page from its starting index in the Origin to the end of the Dest
        dragDest.pages.push(dragOrig.pages[e.oldIndex])
        dragOrig.pages.splice(e.oldIndex, 1);
      }
    },
  })
}

function removePage(pageGroup: PageGroup, index: number) {
  pageGroup.pages.splice(index, 1)
}
function splitDoc(pageGroup: PageGroup, index: number) {
  var remainingPages = pageGroup.pages.splice(index + 1)
  addPageGroup({name: `x_${pageGroup.name}`, pages: remainingPages})
}
function removeDoc(index: number) {
  allPageGroups.value.splice(index, 1)
}
async function downloadDoc(pageGroup: PageGroup) {
  const newPdf = await PDFDocument.create()

  var copiedPages = await newPdf.copyPages(pdfDataDocument, pageGroup.pages.map(({ pageNumber }) => (pageNumber - 1)))
  copiedPages.forEach((p, i) => {
    p.setRotation(degrees(pageGroup.pages[i].rotation))
    newPdf.addPage(p)
  });

  window.open(await newPdf.saveAsBase64({ dataUri: true }), '_blank');
}

</script>

<style>
body {
  background-color: whitesmoke;
}

.pdf-document {
  background-color: lightgray;
  width: 60vw;
  margin: 5px;
  padding-top: 5px;
  padding-bottom: 5px;
}
.pdf-document-header {
  padding: 6px;
  color: black;
  display: flex;
  justify-content: left;
  align-items: center;
}
.pdf-document-header .filename-input {
  width: 100%;
  margin-left: var(--sp-2);
  margin-right: var(--sp-2);
}
.pdf-document-empty {
  margin-top: 10px;
  background-color: var(--gray-50);
  border: 2px dashed var(--gray-300);
  color: var(--gray-400);
  font-size: 14px;
  font-weight: 500;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100px;
}
.pdf-document-body {
  background-color: lightgray;
  display: flex;
  flex-wrap: wrap;
  justify-content: left;
  align-items: center;
  min-height: 75px;
  margin: 10px;
  padding: 0px 10px 10px 10px;
}
.pdf-document-page-group {
  display: flex;
  flex-wrap: wrap;
  justify-content: left;
}
.pdf-document-page-group .pdf-document-page {
  cursor: grab;
  margin-top: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.pdf-document-page-group .pdf-document-page .pdf-viewer {
  border: 2px solid grey;
}
.pdf-document-page-group .pdf-document-page:not(:hover) .page-buttons {
  display: none;
}
.pdf-document-page-group .pdf-document-page .page-buttons {
  position: absolute;
  background-color: rgba(95, 97, 110, 0.7);
  border-radius: 0.5em;
  padding: 3px;
}
.pdf-document-page-group .pdf-document-split {
  cursor: col-resize;
  margin-top: 10px;
  padding-left: 6px;
  padding-right: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.pdf-document-page-group .pdf-document-split:not(:hover) .dashed-line {
  display: none;
}
.pdf-document-page-group .pdf-document-split .dashed-line {
  border-right: 1px dashed black;
  height: 85%;
}
.pdf-document-page-group .pdf-document-split:hover .dashed-spacer {
  display: none;
}
.pdf-document-page-group .pdf-document-split .dashed-spacer {
  border-right: 1px solid lightgray;
}
.pdf-document-drop {
  background-color: var(--gray-200);
}

</style>
