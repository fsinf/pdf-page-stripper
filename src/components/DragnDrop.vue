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

<script lang="ts">
import { defineComponent } from "vue";

export default defineComponent({
  data() {
    return {
      isDragover: false,
    };
  },
  methods: {
    onChange() {
      const fileElem = this.$refs.file as HTMLInputElement;
      const filelist = fileElem.files;
      this.$emit("filesSelected", filelist);
    },
    dragenter(e: DragEvent) {
      this.isDragover = true;
    },
    dragleave(e: DragEvent) {
      this.isDragover = false;
    },
    drop(e: DragEvent) {
      e.preventDefault(); // prevent from open files in new tab
      this.isDragover = false;
      this.$refs.file.files = e.dataTransfer.files;
      this.onChange();
    },
  },
});
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