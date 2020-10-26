<template>
  <form action="#">
    <h3>PDF Stripper</h3>
    <div class="file-field input-field">
      <div class="btn">
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
import { PDFDocument, PDFName } from "pdf-lib";

function usePdfLib() {
  async function stripPdf(bytes: ArrayBuffer) {
    const pdfFile = await PDFDocument.load(bytes);
    const pageLabels = pdfFile.catalog.get(PDFName.of("PageLabels"));
    const pageNumbers = (pageLabels as any).get(PDFName.of("Nums"))
      .array as any[];

    const pagesToKeep = new Set<number>();
    for (let i = 0; i < pageNumbers.length; i++) {
      const element = pageNumbers[i];
      if (element.numberValue !== undefined) {
        // Page number - 1 ==> index
        pagesToKeep.add(element.numberValue - 1);
      }
    }

    const pageCount = pdfFile.getPageCount();
    // NOTE: pageCount - 2, because we always want to keep the last page
    for (let i = pageCount - 2; i >= 0; i--) {
      if (!pagesToKeep.has(i)) {
        pdfFile.removePage(i);
      }
    }
    return pdfFile;
  }

  return { stripPdf };
}

export default defineComponent({
  setup() {
    const pdfLib = usePdfLib();
    const pdfFiles = ref<{ id: string; name: string; result?: string }[]>([]);

    async function onFilesSelected(event: InputEvent) {
      pdfFiles.value.length = 0;
      const files = (event.target as HTMLInputElement).files;

      for (let i = 0; i < files.length; i++) {
        const file = files.item(i);
        pdfFiles.value.push({
          id: "" + i,
          name: `stripped-${file.name}`,
        });
      }

      for (let i = 0; i < files.length; i++) {
        const file = files.item(i);
        const result = new Blob(
          [await (await pdfLib.stripPdf(await file.arrayBuffer())).save()],
          { type: "application/pdf" }
        );
        pdfFiles.value[i].result = window.URL.createObjectURL(result);
      }
    }

    return {
      onFilesSelected,
      pdfFiles,
    };
  },
});
</script>
