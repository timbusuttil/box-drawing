<template>
  <!-- <BoxRenderer :width="7" :startData="startData" /> -->
  <BoxCanvas :width="21" :height="10" />
</template>

<script>
// import BoxRenderer from './components/BoxRenderer.vue'
import BoxCanvas from './components/BoxCanvas.vue'

export default {
  name: 'App',
  components: {
    // BoxRenderer
    BoxCanvas
  },
  // data() {
  //   return {
  //     startData: [
  //       0, 0, 0, 0, 0, 0, 0,
  //       0, 1, 1, 1, 1, 1, 0,
  //       0, 1, 1, 1, 1, 1, 0,
  //       0, 1, 1, 1, 1, 1, 0,
  //       0, 1, 1, 1, 1, 1, 0,
  //       0, 1, 1, 1, 1, 1, 0,
  //       0, 0, 0, 0, 0, 0, 0,
  //     ]
  //   }
  // }
}
</script>

<style>
</style>
