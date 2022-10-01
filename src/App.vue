<template>
  <BoxCanvas :initWidth="29" :initHeight="11" />
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
html, body, #app {
  box-sizing: border-box;
  margin: 0;
  width: 100vw;
  height: 100vh;
}
</style>
