<template>
  <button @click="() => addPageGroup({})">New Page</button>
  <input type="range" min=".15" max="1" step=".05" v-model="scale" class="slider">
  <div class="pdf-document" v-for="(pageGroup, index) in allPageGroups" :id="`new-pdf-document-${index}`" :key="index"
    :ref="(el) => pageGroup.pageElRef = el" :class="{ 'pdf-document-drop': pageGroup.hover }">
    <div class="pdf-document-header">
      <button @click="() => removeDoc(pageGroup, index)">Delete</button>
      <input type="text" :value="pageGroup.name" />
      <button @click="() => downloadDoc(pageGroup, index)">Download</button>
    </div>
    <div v-if="pageGroup.pages.length === 0" class="pdf-document-new">
      <p>Drag Here to Create</p>
    </div>

    <div v-for="(page, index) in pageGroup.pages" :key="page?.pageNumber" class="pdf-document-page">
      <VuePDF :pdf="pdfPageViewer" :page="page.pageNumber" :scale="scale" :rotation="page.rotation" />
      <button @click="() => page.rotation -= 90"><</button><button @click="() => page.rotation += 90">></button><button
            @click="removePage(pageGroup, index)">x</button>
          <p>{{ page.pageNumber }}</p>
    </div>
  </div>
  <iframe :src="iframeSrc" style="width: 100%; height: 100%;"></iframe>
</template>

<script lang="ts" setup>
import * as PDFJS from 'pdfjs-dist';
import LegacyWorker from 'pdfjs-dist/legacy/build/pdf.worker.min?url';
PDFJS.GlobalWorkerOptions.workerSrc = LegacyWorker
import { PDFDocument } from 'pdf-lib'
import { nextTick, watchEffect, reactive, Reactive, ref, Ref, onMounted, watch } from 'vue'
import { VuePDF } from '@tato30/vue-pdf'
import { useDropZone, watchOnce } from '@vueuse/core'
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
  pageElRef: HTMLElement;
};

function addPageGroup(pg: PageGroup | any) {
  allPageGroups.value.push({
    pages: [],
    name: "",
    ...pg,
    hover: false,
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
  nextTick(() => {
    allPageGroups.value.forEach(initDragDrop);
  })
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
      console.log("onDrop.hover", pageGroup.name)
    },
    onOver: () => {
      nextTick(() => {
        pageGroup.hover = true
        console.log("onOver.hover", pageGroup.name)
      })
    },
    onLeave: () => {
      nextTick(() => {
        pageGroup.hover = false
        console.log("onLeave.hover", pageGroup.name)
      })
    }
  })
}
function initUseSortable(pageGroup: PageGroup) {
  console.log("initUseSortable", pageGroup.pages.map(({ pageNumber }) => (pageNumber)))
  useSortable(pageGroup.pageElRef, pageGroup.pages, {
    onStart: (e) => {
      dragOrig = pageGroup
      dragDest = pageGroup
    },
    onUpdate: (e) => {
      // only update the sortable order if the drop is in the same document
      console.log("onUpdate")
      if (dragOrig !== undefined && dragDest !== undefined && dragOrig === dragDest
        && e.oldIndex !== undefined && e.newIndex !== undefined) {
        moveArrayElement(pageGroup.pages, e.oldIndex, e.newIndex)
        console.log("onUpdate", pageGroup.pages.map(({ pageNumber }) => (pageNumber)))
      }
    },
    onEnd: (e) => {
      if (dragOrig !== undefined && dragDest !== undefined
        && dragOrig !== dragDest && e.oldIndex !== undefined) {
          console.log("onEnd")
        // move the page from its starting index in the Origin to the end of the Dest
        const pageArrayIndex = e.oldIndex - 1
        if (dragOrig.pages[pageArrayIndex]?.pageNumber > 0) {
          dragDest.pages.push(dragOrig.pages[pageArrayIndex])
          dragOrig.pages.splice(pageArrayIndex, 1)
          console.log("onEnd", pageGroup.pages.map(({ pageNumber }) => (pageNumber)))
        }

      }
      // Add new pageGroup
      // if (allPageGroups.filter((a) => a.pages.length === 0).length === 0) {
      //   addEmptyPageGroup()
      // }
    },
  })
}

function removePage(pageGroup: PageGroup, index: Number) {
  pageGroup.pages.splice(index, 1)
}
function removeDoc(pageGroup: PageGroup, index: Number) {
  pageGroup.splice(index, 1)
}
async function downloadDoc(pageGroup: PageGroup) {

  const newPdf = await PDFDocument.create()

  // console.log(newPdf.getPageCount())
  // for (const page of pageGroup.pages) {
  //   var pageIndex = page.pageNumber - 1
  //   console.log("num", page.pageNumber, pageIndex)
  //   var [copiedPage] = await newPdf.copyPages(getPdfDocument(page._pdfKey), [pageIndex])
  //   newPdf.addPage(copiedPage)
  // }
  // console.log(newPdf.getPageCount())

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
  display: flex;
  flex-wrap: wrap;
  justify-content: left;
  align-items: center;
  width: 80vw;
  min-height: 75px;
  margin: 5px;
}

.pdf-document .pdf-document-header {
  border: 1px solid grey;
  padding: 3px;
  color: black;
  display: flex;
  justify-content: left;
  width: 100%;
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
