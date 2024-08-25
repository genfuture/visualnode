<template>
  <div id="app">
    <HistogramChart />
  </div>
</template>

<script>
import HistogramChart from './components/HistogramChart.vue';

export default {
  name: 'App',
  components: {
    HistogramChart,
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
