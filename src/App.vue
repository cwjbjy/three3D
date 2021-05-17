<template>
  <div id="app">
    <space/>
  </div>
</template>

<script>
import space from './components/space.vue'

export default {
  name: 'App',
  components: {
    space
  }
}
</script>

<style>
body{
  margin: 0;
  padding: 0;
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>
