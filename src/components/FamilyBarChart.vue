<template>
  <div class="bg-white rounded-lg shadow-md p-4 sm:p-6 border border-gray-200">
    <h3 class="text-lg font-medium text-gray-900 mb-4">Family Sizes</h3>
    <div ref="tooltipContainer" class="relative">
      <svg ref="svgRef" class="w-full h-[400px]"></svg>
      <div v-if="tooltip" class="absolute bg-black text-white px-2 py-1 rounded-md text-sm z-10 pointer-events-none shadow-lg"
           :style="tooltipStyle">
        {{ tooltip.text }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, nextTick, computed } from 'vue'
import * as d3 from 'd3'

const props = defineProps({
  families: { type: Array, default: () => [] },
  onSelect: { type: Function, default: null }
})

const emit = defineEmits(['select'])
const svgRef = ref(null)
const tooltipContainer = ref(null)
const tooltipData = ref(null)

const tooltipStyle = computed(() => {
  if (!tooltipData.value || !tooltipContainer.value) return {}
  const rect = tooltipContainer.value.getBoundingClientRect()
  let x = tooltipData.value.mouseX - rect.left
  let y = tooltipData.value.mouseY - rect.top

  // Approximate tooltip dimensions
  const tooltipWidth = 150
  const tooltipHeight = 30

  // Adjust to prevent overflow
  if (x + tooltipWidth > rect.width) {
    x = rect.width - tooltipWidth - 10
  }
  if (y - tooltipHeight < 0) {
    y = tooltipHeight + 10
  } else {
    y -= tooltipHeight + 10 // Position above mouse by default
  }

  // Center horizontally
  x -= tooltipWidth / 2

  return {
    left: `${Math.max(0, x)}px`,
    top: `${y}px`,
    transform: 'translateX(-50%)'
  }
})

watch(() => props.families, async (families) => {
  if (families && families.length > 0) {
    await nextTick()
    renderChart(families)
  }
}, { immediate: true })

const renderChart = (families) => {
  d3.select(svgRef.value).selectAll('*').remove()

  const svg = d3.select(svgRef.value)
    .attr('width', 600)
    .attr('height', 400)
    .attr('viewBox', `0 0 600 400`)
    .style('max-width', '100%')
    .style('height', 'auto')

  const margin = { top: 20, right: 40, bottom: 80, left: 80 }
  const innerWidth = 600 - margin.left - margin.right
  const innerHeight = 400 - margin.top - margin.bottom

  const xScale = d3.scaleBand()
    .domain(families.map(f => f.name))
    .range([margin.left, 600 - margin.right])
    .padding(0.1)

  const yScale = d3.scaleLinear()
    .domain([0, d3.max(families, f => f.count)])
    .range([innerHeight + margin.top, margin.top])

  const g = svg.append('g')
    .attr('transform', `translate(0, 0)`)

  // Bars
  const bars = g.selectAll('rect')
    .data(families)
    .join('rect')
    .attr('x', d => xScale(d.name))
    .attr('y', d => yScale(d.count))
    .attr('width', xScale.bandwidth())
    .attr('height', d => innerHeight - (yScale(d.count) - margin.top))
    .attr('fill', '#0066eb')
    .attr('opacity', 0.7)
    .on('mouseover', function(event, d) {
      tooltipData.value = {
        text: `${d.name}: ${d.count} members`,
        mouseX: event.clientX,
        mouseY: event.clientY
      }
      d3.select(this).transition().duration(200).attr('opacity', 0.9)
    })
    .on('mouseout', () => {
      tooltipData.value = null
      bars.transition().duration(200).attr('opacity', 0.7)
    })
    .on('click', (event, d) => {
      if (props.onSelect) {
        props.onSelect(d.name)
      } else {
        emit('select', d.name)
      }
    })

  // X axis
  g.append('g')
    .attr('transform', `translate(0, ${innerHeight + margin.top})`)
    .call(d3.axisBottom(xScale))
    .selectAll('text')
    .style('text-anchor', 'end')
    .attr('dx', '-.8em')
    .attr('dy', '.15em')
    .attr('transform', 'rotate(-45)')

  // Y axis
  g.append('g')
    .attr('transform', `translate(${margin.left}, 0)`)
    .call(d3.axisLeft(yScale))

  // Labels
  g.append('text')
    .attr('x', 300)
    .attr('y', 390)
    .attr('text-anchor', 'middle')
    .text('Family Name')
    .attr('fill', '#212529')

  g.append('text')
    .attr('transform', 'rotate(-90)')
    .attr('y', 0 - margin.left + 20)
    .attr('x', 0 - (innerHeight / 2))
    .attr('dy', '1em')
    .text('Number of Members')
    .attr('fill', '#212529')
}
</script>