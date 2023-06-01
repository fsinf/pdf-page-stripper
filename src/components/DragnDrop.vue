<template>
  <div
    class="dropzone"
    :class="[isDragover ? 'dragover' : '']"
    @dragover="dragenter"
    @dragleave="dragleave"
    @dragover.prevent
    @drop="drop"
  >
    <label class="label">
      Drag'n drop or <strong class="selectSpan">select</strong> PDF-files here
      <input
        type="file"
        name="file"
        class="fileinput"
        accept="application/pdf,.pdf"
        multiple
        @change="onChange()"
        ref="file"
      />
    </label>
  </div>
</template>

<script setup lang="ts">
import { ref } from "vue";

const emits = defineEmits<{
  (event: "filesSelected", files: FileList): void;
}>();

// ref wraps them in proxy objects
const isDragover = ref(false);
const file = ref<HTMLInputElement | null>(null);

function onChange() {
  const filelist = file.value?.files;
  if (!filelist || filelist.length === 0) return;
  emits("filesSelected", filelist);
}
function dragenter(e: DragEvent) {
  isDragover.value = true;
}
function dragleave(e: DragEvent) {
  isDragover.value = false;
}
function drop(e: DragEvent) {
  // prevent from opening files in new tab
  e.preventDefault();
  isDragover.value = false;
  if (file.value === null) return;
  file.value.files = e.dataTransfer?.files ?? null;
  onChange();
}

// all variables and functions are automatically exposed to the template
</script>

<style scoped>
.dropzone {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 80px;
  transition: 1s;
}
.dragover {
  /* background-color: #3e9def; */
  background-color: #0011c2;
  transform: scale(1.1);
  color: #fff;
}

.selectSpan:hover {
  cursor: pointer;
  text-decoration: underline;
}

.fileinput {
  visibility: hidden;
  position: absolute;
}
</style>
