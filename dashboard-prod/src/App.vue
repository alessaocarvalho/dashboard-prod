<template>
  <div id="app">
    <ProdDashboard />
  </div>
</template>

<script>
import ProdDashboard from './components/ProdDashboard.vue';

export default {
  name: 'App',
  components: {
    ProdDashboard,
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 20px;
}
</style>