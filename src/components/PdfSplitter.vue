<template>
  <button @click="() => addPageGroup({})">New Page</button>
  <input type="range" min=".15" max="1" step=".05" v-model="scale" class="slider">
  <div class="pdf-document" v-for="(pageGroup, index) in allPageGroups" :id="`new-pdf-document-${index}`" :key="index"
     :class="{ 'pdf-document-drop': pageGroup.hover }">
    <div class="pdf-document-header">
      <button @click="() => downloadDoc(pageGroup)">Download</button>
      <input type="text" style="width: 100%;" :value="pageGroup.name" />
      <button @click="() => removeDoc(index)">Delete</button>
    </div>
    <div :ref="(el) => pageGroup.pageElRef = el" class="pdf-document-body">
      <div v-if="pageGroup.pages.length === 0" class="pdf-document-new">
        <p>Drag Here to Create</p>
      </div>
      <div v-for="(page, index) in pageGroup.pages" :key="page.pageNumber" class="pdf-document-page">
        <p>PAGE</p>
        <VuePDF :pdf="pdfPageViewer" :page="page.pageNumber" :scale="scale" :rotation="page.rotation" />
        <button @click="() => page.rotation -= 90"><</button><button @click="() => page.rotation += 90">></button><button
              @click="removePage(pageGroup, index)">x</button>
            <p>{{ page.pageNumber }} | {{ index }}</p>
      </div>
    </div>
  </div>
  <iframe :src="iframeSrc" style="width: 100%; height: 100%;"></iframe>
</template>

<script lang="ts" setup>
import * as PDFJS from 'pdfjs-dist';
import LegacyWorker from 'pdfjs-dist/legacy/build/pdf.worker.min?url';
PDFJS.GlobalWorkerOptions.workerSrc = LegacyWorker
import { PDFDocument } from 'pdf-lib'
import { nextTick, Reactive, ref, Ref, onMounted } from 'vue'
import { VuePDF } from '@tato30/vue-pdf'
import { useDropZone } from '@vueuse/core'
import { useSortable, moveArrayElement } from '@vueuse/integrations/useSortable'

const props = defineProps({
  preLoadPdfUrls: Array<string>,
})

const pdfPageViewer:Ref<PDFJS.PDFDocumentLoadingTask> = ref()
var pdfDataDocument:PDFDocument

async function addPdfPagesFromUrl(url: string): Promise<number> {
  const arrayBuffer = await fetch(url).then(res => res.arrayBuffer())
  return await addPdfPages(arrayBuffer)
}

async function addPdfPages(arrayBuffer:ArrayBuffer): Promise<number> {
  var pdfDocument = await PDFDocument.load(arrayBuffer)
  var copiedPages = await pdfDataDocument.copyPages(pdfDocument, numArray(pdfDocument.getPageCount()))
  copiedPages.forEach((page) => {
    pdfDataDocument.addPage(page)
  })
  return copiedPages.length
}

async function renderPdfPages() {
  const loadingTask = PDFJS.getDocument(await pdfDataDocument.save())
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
  var loadedDocs = await Promise.all(props.preLoadPdfUrls.map(async (url) => {
    var numOfPages = await addPdfPagesFromUrl(url)
    return {
      url,
      numOfPages
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
      name: url,
    })
  })
  addPageGroup({})
})

var dragOrig: PageGroup | undefined
var dragDest: PageGroup | undefined

function initDragDrop(pg: PageGroup) {
  if (pg.pageElRef) {
    console.log('Init drag drop', pg.name)
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
      // Add new pageGroup
      if (allPageGroups.value.filter((a) => a.pages.length === 0).length === 0) {
        addPageGroup({})
      }
    },
  })
}

function removePage(pageGroup: PageGroup, index: Number) {
  pageGroup.pages.splice(index, 1)
}
function removeDoc(index: Number) {
  allPageGroups.value.splice(index, 1)
}
async function downloadDoc(pageGroup: PageGroup) {

  const newPdf = await PDFDocument.create()

  var copiedPages = await newPdf.copyPages(pdfDataDocument, pageGroup.pages.map(({pageNumber}) => (pageNumber - 1)))
  copiedPages.forEach(p => newPdf.addPage(p));

  console.log("loading")
  iframeSrc.value = await newPdf.saveAsBase64({ dataUri: true });
  console.log("loaded")

}

const scale = ref(.20)
const iframeSrc = ref('')

</script>

<style>
body {
  background-color: whitesmoke;
}

.pdf-document {
  background-color: lightgray;
  width: 80vw;
  margin: 5px;
}

.pdf-document-header {
  border: 1px solid grey;
  padding: 3px;
  color: black;
  display: flex;
  justify-content: left;
  width: 100%;
}

.pdf-document-new {
  border: 1px dashed grey;
  color: black;
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
  margin: 5px;
}

.pdf-document-page {
  cursor: grab;
  background-color: grey;
  padding: 3px;
  margin: 3px;
}

.pdf-document-drop {
  background-color: var(--gray-200);
}
</style>
