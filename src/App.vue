<template>
  <div id="app">
    <img alt="Bear logo" src="./assets/logo.png">
    <Webcam />
  </div>
</template>

<script>

import Webcam from './components/Webcam.vue';

export default {
  name: 'App',
  components: {
    Webcam
  },
  data: function() {
    return {
      connection: null
    }
  },
  methods: {
  },
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
