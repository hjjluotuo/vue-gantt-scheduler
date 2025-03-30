<template>
  <div id="app">
    <h1>Gantt Scheduler</h1>
    <GanttChart :tasks="tasks" />
  </div>
</template>

<script>
import GanttChart from './components/GanttChart.vue';

export default {
  name: 'App',
  components: {
    GanttChart,
  },
  data() {
    return {
      tasks: [
        { name: 'Task 1', start: '2023-10-01', end: '2023-10-05' },
        { name: 'Task 2', start: '2023-10-05', end: '2023-10-10' },
        { name: 'Task 3', start: '2023-10-10', end: '2023-10-15' },
      ],
    };
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
