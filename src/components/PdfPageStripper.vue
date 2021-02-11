<template>
  <form action="#">
    <h3>PDF Stripper</h3>
    <div class="file-field input-field">
      <div class="btn card">
        <span>PDFs</span>
        <input
          type="file"
          accept="application/pdf,.pdf"
          multiple
          @change="onFilesSelected($event)"
        />
      </div>
    </div>
  </form>

  <div>
    <h3>Stripped PDFs</h3>
    <p v-if="zipping.ready == 0"> ZIP in process ... </p>
    <a v-if="zipping.ready == 1" download="stripped-pdf" :href="zipping.result"> Download as ZIP </a>
    <ul>
      <li v-for="item in pdfFiles" :key="item.id">
        {{ item.name }}
        <a v-if="item.result" :download="item.name" :href="item.result"
          >Download</a
        >
      </li>
    </ul>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from "vue";
import { PDFArray, PDFDict, PDFDocument, PDFName, PDFNumber, PDFObject } from "pdf-lib";
import JSZip from "jszip";

interface PDFResult { id: string; name: string; result?: string };

function usePdfLib() {
  /**
   * It finds the index of the last page of each defined page label of the PDF document. The only the found pages are returned.
   * 
   * More information about page labels:
   * https://www.adobe.com/content/dam/acom/en/devnet/pdf/pdfs/pdf_reference_archives/PDFReference.pdf#page=501&zoom=100,118,506
   * @returns PDFDocument with last pages of each label range
   */
  async function stripPdf(bytes: ArrayBuffer) {
    const pdfFile = await PDFDocument.load(bytes);
    const pageLabels = <PDFDict> pdfFile.catalog.get(PDFName.of("PageLabels"));
    if (!pageLabels) return;
    const pageNumbers = <PDFObject[]>(<PDFArray> pageLabels.get(PDFName.of("Nums"))).asArray();
    if (!pageNumbers) return;

    const pagesToKeep = new Set<number>();
    for (let i = 1; i < pageNumbers.length; i++) {
      const element = pageNumbers[i];
      if(element instanceof PDFNumber) {
        // Page number - 1 ==> index
        // each page number == first index of label range ... page number - 1 == last index of previous label range
        pagesToKeep.add(element.asNumber() - 1);
      }
    }

    let deletedAnyPage = false;
    const pageCount = pdfFile.getPageCount();
    // NOTE: pageCount - 2, because we always want to keep the last page (since it is not included in the set)
    for (let i = pageCount - 2; i >= 0; i--) {
      if (!pagesToKeep.has(i)) {
        pdfFile.removePage(i);
        deletedAnyPage = true;
      }
    }
    if (!deletedAnyPage) return;
    return pdfFile;
  }

  /**
   * Converts all striped PDF to one zip file.
   * 
   * @returns URL to blob(with zip file) or undefined if less than 2 pdf got striped
   */
  async function zipFiles(files: PDFResult[]) {
    const zip = new JSZip();

    for(let i = 0; i < files.length; i++){
      if(files[i].result === undefined)
        continue;
      const blob = await fetch(files[i].result).then(r => r.blob());
      zip.file(files[i].name, blob);
    }

    const resZip = await zip.generateAsync<"blob">({type: "blob", compression: "DEFLATE"});
    return window.URL.createObjectURL(resZip);
  }

  return { stripPdf, zipFiles };
}

export default defineComponent({
  setup() {
    const pdfLib = usePdfLib();
    const pdfFiles = ref<PDFResult[]>([]);

    let zipping = ref<{ ready: number, result?: string}>({ready: -1});

    async function onFilesSelected(event: InputEvent) {
      pdfFiles.value.length = 0;
      const files = (event.target as HTMLInputElement).files;

      for (let i = 0; i < files.length; i++) {
        const file = files.item(i);

        pdfFiles.value.push({
          id: "" + i,
          name: `stripped-${file.name}`,
        });
        
        const strippedPdf = await pdfLib.stripPdf(await file.arrayBuffer());
        if (strippedPdf) {
          const result = new Blob([await strippedPdf.save()], {
            type: "application/pdf",
          });
          pdfFiles.value[i].result = window.URL.createObjectURL(result);

          zipping.value.ready = 0;
        } else {
          pdfFiles.value[i].name += " - found nothing to strip";
        }
      }

      if(zipping.value.ready === 0) {
        zipping.value.result = await pdfLib.zipFiles(pdfFiles.value);
        zipping.value.ready = 1;
      }
    }

    return {
      onFilesSelected,
      pdfFiles,
      zipping,
    };
  },
});
</script>
