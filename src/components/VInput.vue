<template>
  <input
    type="text"
    class="flex-1 px-2 py-1 focus:outline-none rounded-md text-black"
    v-model="value"
  />
</template>

<script>
export default {
  name: "v-input",
  props: ["modelValue"],
  emits: ["update:modelValue"],
  computed: {
    value: {
      get() {
        return this.modelValue;
      },
      set(value) {
        this.$emit("update:modelValue", value);
      },
    },
  },
};
</script>

<style></style>
