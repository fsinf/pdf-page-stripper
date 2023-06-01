<template>
  <h2>PDF Stripper</h2>

  <div class="artboard">
    <div class="card">
      <div
        :class="[processDone == 100 ? 'card__side--back_active' : '']"
        class="card__side card__side--back"
      >
        <DownloadArea
          :pdfFiles="pdfFiles"
          :zip="zipping"
          @reset="() => (processDone = 0)"
        />
      </div>

      <div
        :class="[
          processDone == 100 ? 'card__side--front_active' : '',
          processDone > 0 ? 'toProgressbar' : '',
        ]"
        class="card__side card__side--front"
      >
        <div
          class="progress-done"
          :class="[processDone == 0 ? 'hideProgressBar' : '']"
          :style="{ width: processDone + '%' }"
        ></div>
        <DragnDrop
          :class="[processDone > 0 ? 'hideDragnDrop' : '']"
          @filesSelected="onFilesSelected"
        />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from "vue";
import {
  PDFArray,
  PDFDict,
  PDFDocument,
  PDFName,
  PDFNumber,
  PDFObject,
} from "pdf-lib";
import * as JSZip from "jszip";

import DragnDrop from "./DragnDrop.vue";
import DownloadArea from "./DownloadArea.vue";

interface PDFResult {
  id: string;
  name: string;
  result?: string;
}

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
    const pageLabels = <PDFDict>pdfFile.catalog.get(PDFName.of("PageLabels"));
    if (!pageLabels) return;
    const pageNumbers = <PDFObject[]>(
      (<PDFArray>pageLabels.get(PDFName.of("Nums"))).asArray()
    );
    if (!pageNumbers) return;

    const pagesToKeep = new Set<number>();
    for (let i = 1; i < pageNumbers.length; i++) {
      const element = pageNumbers[i];
      if (element instanceof PDFNumber) {
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

    for (let i = 0; i < files.length; i++) {
      if (files[i].result === undefined) continue;
      const blob = await fetch(files[i].result).then((r) => r.blob());
      zip.file(files[i].name, blob);
    }

    const resZip = await zip.generateAsync<"blob">({
      type: "blob",
      compression: "DEFLATE",
    });

    return { url: window.URL.createObjectURL(resZip), size: resZip.size };
  }

  return { stripPdf, zipFiles };
}

export default defineComponent({
  setup() {
    const pdfLib = usePdfLib();
    const pdfFiles = ref<PDFResult[]>([]);

    const zipping = ref<{ state: number; result?: string; size?: number }>({
      state: -1,
    });
    const processDone = ref<number>(0);

    async function onFilesSelected(files: FileList) {
      processDone.value = 0.1;
      pdfFiles.value.length = 0;
      zipping.value.state = -1;

      if (files.length == 0) return;

      for (let i = 0; i < files.length; i++) {
        const file = files.item(i);
        if (!file) continue;

        pdfFiles.value.push({
          id: "" + i,
          name: `stripped-${file.name}`,
        });

        const fileBytes = await file.arrayBuffer();
        processDone.value = ((i + 0.3) / files.length) * 100;

        const strippedPdf = await pdfLib.stripPdf(fileBytes);
        processDone.value = ((i + 0.7) / files.length) * 100;

        if (strippedPdf) {
          const result = new Blob([await strippedPdf.save()], {
            type: "application/pdf",
          });
          pdfFiles.value[i].result = window.URL.createObjectURL(result);

          zipping.value.state = 0;
        } else {
          pdfFiles.value[i].name += " - found nothing to strip";
        }

        processDone.value = ((i + 1) / files.length) * 100;
      }
      processDone.value = 100;

      if (zipping.value.state === 0) {
        const res = await pdfLib.zipFiles(pdfFiles.value);
        zipping.value.result = res.url;
        zipping.value.size = res.size;
        zipping.value.state = 1;
      }
    }

    return {
      onFilesSelected,
      pdfFiles,
      zipping,
      processDone,
    };
  },
  components: {
    DragnDrop,
    DownloadArea,
  },
});
</script>

<style scoped>
/* Reset */
*,
*::after,
*::before {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html {
  font-size: 62.5%;
}
@media (max-width: 75em) {
  html {
    font-size: 56.25%;
  }
}
@media (max-width: 56.25em) {
  html {
    font-size: 50%;
  }
}
@media (min-width: 112.5em) {
  html {
    font-size: 75%;
  }
}

body {
  height: 100vh;
  background-color: #ece0e8;
  box-sizing: border-box;
}

.progress-done {
  background: rgb(86, 204, 242);
  /* background: linear-gradient(90deg, rgba(86,204,242,1) 0%, rgba(47,128,237,1) 100%); */
  background: linear-gradient(
    90deg,
    rgba(47, 128, 237, 1) 0%,
    rgba(0, 17, 194, 1) 100%
  );
  border-radius: 10px;
  height: 100%;
  transition: 300ms;
}
.hideProgressBar {
  visibility: hidden;
  position: absolute;
}
.hideDragnDrop {
  color: transparent;
}

/* Card stylings */
.artboard {
  display: flex;
  flex-flow: row;
  align-items: center;
  justify-content: center;
  padding: 1rem;
  height: 100%;
  position: relative;
}
@media (max-width: 37.5em) {
  .artboard {
    padding: 1rem;
  }
}

.card {
  flex: initial;
  position: relative;
  min-width: 30rem;
  width: 70%;
  -moz-perspective: 200rem;
  perspective: 200rem;
  margin: 2rem;
}
.card__side {
  transition: all 0.8s ease;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  -webkit-backface-visibility: hidden;
  backface-visibility: hidden;
  border-radius: 15px;
  overflow: hidden;
  box-shadow: 0 2rem 6rem rgba(0, 0, 0, 0.15);
}

.card__side--back {
  background-color: #fff;
  transform: rotateY(180deg);
}

.card__side--front {
  height: 75px;
}

.toProgressbar {
  margin-top: 25px;
  height: 25px;
  box-shadow: 0.3rem 0.4rem 2rem rgba(0, 0, 0, 0.25);
}

.card__side--front_active {
  transform: rotateY(-180deg);
}
.card__side--back_active {
  transform: rotateY(0deg);
}
</style>
