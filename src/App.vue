<template>
  <div id="app">
    <Drawing msg="Welcome to Your Vue.js App"/>
  </div>
</template>

<script>
import Drawing from './components/drawing.vue'

export default {
  name: 'App',
  components: {
    Drawing
  }
}
</script>

<style>
*{margin: 0;padding: 0;box-sizing: border-box;}
*::before{box-sizing: border-box;}
*::after{box-sizing: border-box;}
ul{padding: 0; }
li{list-style: none;}
.clearfix::after{
    content: '';
    display: block;
    clear: both;
}
</style>
