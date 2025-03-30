<template>
  <div ref="gantt" class="gantt-chart"></div>
</template>

<script>
import * as d3 from 'd3';
import draggable from 'vuedraggable';

export default {
  components: {
    draggable,
  },
  props: {
    tasks: {
      type: Array,
      required: true,
    },
  },
  data() {
    return {
      width: 800,
      height: 400,
      margin: { top: 20, right: 30, bottom: 30, left: 40 },
    };
  },
  mounted() {
    this.drawGantt();
  },
  methods: {
    drawGantt() {
      const { width, height, margin } = this;
      const svgWidth = width - margin.left - margin.right;
      const svgHeight = height - margin.top - margin.bottom;

      const svg = d3.select(this.$refs.gantt)
        .append('svg')
        .attr('width', width)
        .attr('height', height)
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`);

      const xScale = d3.scaleTime()
        .domain(d3.extent(this.tasks, d => new Date(d.start)))
        .range([0, svgWidth]);

      const yScale = d3.scaleBand()
        .domain(this.tasks.map(d => d.name))
        .range([0, svgHeight])
        .padding(0.1);

      svg.append('g')
        .attr('class', 'x-axis')
        .attr('transform', `translate(0,${svgHeight})`)
        .call(d3.axisBottom(xScale));

      svg.append('g')
        .attr('class', 'y-axis')
        .call(d3.axisLeft(yScale));

      svg.selectAll('.bar')
        .data(this.tasks)
        .enter().append('rect')
        .attr('class', 'bar')
        .attr('x', d => xScale(new Date(d.start)))
        .attr('y', d => yScale(d.name))
        .attr('width', d => xScale(new Date(d.end)) - xScale(new Date(d.start)))
        .attr('height', yScale.bandwidth())
        .attr('fill', '#69b3a2')
        .call(draggable);
    },
  },
};
</script>

<style scoped>
.gantt-chart {
  border: 1px solid #ccc;
}
.bar {
  cursor: pointer;
}
</style>
