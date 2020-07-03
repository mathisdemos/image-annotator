<template>
  <div id="app">
    <image-annotator/>
  </div>
</template>

<script>
import ImageAnnotator from './components/ImageAnnotator.vue'

export default {
  name: 'App',
  components: {
    ImageAnnotator
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
