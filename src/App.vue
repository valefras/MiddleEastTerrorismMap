<template>
  <router-view />
</template>
<script>
import { onMounted } from "vue";
export default {
  setup() {
    onMounted(() => {
      document.title = "Terrorism in the Middle East Map";
    });
  },
};
</script>
<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>
