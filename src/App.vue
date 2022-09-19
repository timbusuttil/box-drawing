<template>
  <BoxCanvas :width="21" :height="11" />
  <!-- test data for centering -->
  <!-- <BoxCanvas :width="5" :height="5" :startData="[' ',' ','o','o','o',' ',' ','o','o','o',' ',' ','o','o','o',' ',' ',' ',' ',' ',' ',' ',' ',' ',' ']" /> -->
</template>

<script>
import BoxCanvas from './components/BoxCanvas.vue'

export default {
  name: 'App',
  components: {
    BoxCanvas
  },
}
</script>

<style>
</style>
