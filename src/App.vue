<template>
  <PdfPageStripper />
</template>

<script>
import { defineComponent } from "vue";
import PdfPageStripper from "./components/PdfPageStripper.vue";

export default defineComponent({
  name: "App",
  components: {
    PdfPageStripper,
  },
});
</script>
