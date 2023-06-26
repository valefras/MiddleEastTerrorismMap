<template>
  <div
    class="absolute w-1/4 top-0 right-0 m-6 bg-gray-800 text-gray-100 p-4 rounded-md"
  >
    <div class="font-bold text-2xl text-center mb-2">Attack Info</div>
    <div class="text-gray-400 text-sm">Date</div>
    <div class="mb-2">{{ date }}</div>
    <div class="text-gray-400 text-sm">Summary</div>
    <div class="mb-2">{{ summary }}</div>
    <div class="text-gray-400 text-sm">Number of casualties</div>
    <div class="mb-2">{{ nkill }}</div>
    <div class="text-gray-400 text-sm">Number of wounded</div>
    <div class="mb-2">{{ nwound }}</div>
    <div class="text-gray-400 text-sm">Perpetrators</div>
    <div class="mb-2">{{ gname }}</div>
  </div>
</template>

<script>
import { computed } from "vue";

export default {
  props: {
    attackData: {
      type: Object,
      required: true,
    },
  },
  setup(props) {
    const data = computed(() => {
      return props.attackData;
    });

    const summary = computed(() => {
      if (data.value.summary) {
        return data.value.summary;
      }
      return "Summary unavailable";
    });
    const date = computed(() => {
      return data.value.date;
    });
    const nkill = computed(() => {
      if (data.value.nkill) {
        return data.value.nkill;
      }
      return "Number of casualties unavailable";
    });
    const nwound = computed(() => {
      if (data.value.nwound) {
        return data.value.nwound;
      }
      return "Number of wounded unavailable";
    });
    const gname = computed(() => {
      if (data.value.gname) {
        return data.value.gname;
      }
      return "Group name unavailable";
    });

    return { data, date, summary, nkill, nwound, gname };
  },
};
</script>

<style></style>
