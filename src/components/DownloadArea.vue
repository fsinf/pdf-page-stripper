<template>
  <div>
    <!-- <h3>Stripped PDFs</h3> -->
    <a
      v-if="zip.state == 1"
      download="stripped-pdfs.zip"
      class="face-button"
      :href="zip.result"
    >
      <div class="face-primary">Download as .zip</div>
      <div class="face-secondary">
        Size: {{ Math.round(zip.size / 10000) / 100 }} mb
      </div>
    </a>
    <a v-else-if="zip.state == 0" class="face-button">
      <div class="face-primary">Loading .zip</div>
      <div class="face-secondary">Loading...</div>
    </a>

    <div style="max-height: calc(50vh - 123px); overflow: auto">
      <div class="card__details">
        <ul>
          <li v-for="item in pdfFiles" :key="item.id">
            {{ item.name }}
            <a v-if="item.result" :download="item.name" :href="item.result">
              Download
            </a>
          </li>
        </ul>
      </div>
    </div>

    <a class="resetbtn" @click="$emit('reset')"> Strip again </a>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";

export default defineComponent({
  props: ["pdfFiles", "zip"],
});
</script>


<style scoped>
.card__details {
  font-family: "Inconsolata", monospace;
  padding: 1rem 0rem;
}
.card__details ul {
  list-style: none;
  width: 100%;
  margin: 0 auto;
  padding: 0;
}
.card__details ul li {
  text-align: center;
  font-size: 13px;
  padding: 1rem;
}
.card__details ul li:not(:last-child) {
  border-bottom: 1px solid #eee;
}

/* Button */
.face-button {
  height: 50px;
  display: inline-block;
  border: 3px solid #0011c2;
  /* font-family: "Roboto", sans-serif; */
  /* font-size: 20px; */
  font-weight: 500;
  text-align: center;
  text-decoration: none;
  color: #0011c2;
  overflow: hidden;
}
.face-button .icon {
  margin-right: 6px;
}
.face-button .face-primary,
.face-button .face-secondary {
  display: block;
  padding: 0 32px;
  line-height: 50px;
  transition: margin 0.3s;
}
.face-button .face-primary {
  background-color: #0011c2;
  color: #fff;
}
.face-button:hover .face-primary {
  margin-top: -50px;
}

a.resetbtn {
  display: inline-block;
  padding: 0.35em 1.2em;
  margin: 2rem 0.3em 2.3em;
  border: #0011c2 2px solid;
  border-radius: 0.12em;
  box-sizing: border-box;
  text-decoration: none;
  color: #0011c2;
  text-align: center;
  transition: all 0.2s;
}

a.resetbtn:hover {
  color: #fff;
  background-color: #0011c2;
  cursor: pointer;
}
</style>