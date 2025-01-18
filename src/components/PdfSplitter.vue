<template>
    <div class="pdf-document" v-for="(pages, index) in newPdfs" :key="index"
        :ref="(el) => (initUseDropZoneSortable(index, el))">
        <div v-for="page in pages" :key="page" class="pdf-document-page draggable">
            <VuePDF :pdf="pdf" :page="page" :width="100" />
        </div>
    </div>
</template>

<script lang="ts" setup>
import { nextTick, watch, reactive, Reactive } from 'vue'
import { VuePDF, usePDF } from '@tato30/vue-pdf'
import '@tato30/vue-pdf/style.css'

const { pdf, pages: totalPdfPages } = usePDF('https://mozilla.github.io/pdf.js/web/compressed.tracemonkey-pldi-09.pdf')
import { useDropZone } from '@vueuse/core'
import { useSortable, moveArrayElement } from '@vueuse/integrations/useSortable'

const newPdfs: Reactive<Reactive<number[]>[]> = reactive<Reactive<number[]>[]>([[]])
var dragOrigIndex = 0
var dragDestIndex = 0

// when the PDF has updated the pages, load them into the splitter
watch(totalPdfPages, (p) => {
    newPdfs[0].push(...Array.from({ length: p - 1 }, (_, index) => index + 1))
})

// add a new empty document if one does not exist
watch(newPdfs, (p) => {
    var emptyDocs = p.filter((a) => a.length === 0).length
    if (emptyDocs === 0) {
        nextTick(() => {
            newPdfs.push([])
        })

    }
})

function initUseDropZoneSortable(index: number, el: HTMLElement | any | null) {
    var newPdfPages = newPdfs[index]
    useDropZone(el, {
        onDrop: () => {
            dragDestIndex = index
        },
    })
    useSortable(el, newPdfPages, {
        onStart: () => {
            dragOrigIndex = index
        },
        onUpdate: (e) => {
            // only update the sortable order if the drop is in the same document
            if (dragOrigIndex === dragDestIndex && e.oldIndex !== undefined && e.newIndex !== undefined) {
                moveArrayElement(newPdfs[dragOrigIndex], e.oldIndex, e.newIndex)
            }
        },
        onEnd: (e) => {
            if (dragOrigIndex !== dragDestIndex && e.oldIndex !== undefined) {
                var dragOrig = newPdfs[dragOrigIndex]
                var dragDest = newPdfs[dragDestIndex]

                // move the page from its starting index in the Origin to the end of the Dest
                dragDest.push(dragOrig[e.oldIndex])
                dragOrig.splice(e.oldIndex, 1);

            }
        },
    })

}

</script>

<style>
.draggable {
    cursor: grab;
}

.pdf-document {
    background-color: grey;
    display: flex;
    flex-wrap: wrap;
    justify-content: left;
    align-items: center;
    width: 80vw;
    min-height: 75px;
    margin: 5px;
}

.pdf-document .pdf-document-page {
    background-color: black;
    padding: 3px;
    margin: 5px;
}
</style>
