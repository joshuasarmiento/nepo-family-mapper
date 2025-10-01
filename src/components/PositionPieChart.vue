<template>
  <div class="bg-white rounded-lg shadow-md p-4 sm:p-6 border border-gray-200">
    <h3 class="text-lg font-medium text-gray-900 mb-4">Position Distribution</h3>
    <div ref="tooltipContainer" class="relative">
      <svg ref="svgRef" class="w-full h-[450px]"></svg>
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
  nodes: { type: Array, default: () => [] },
  onSelect: { type: Function, default: null }
})

const emit = defineEmits(['select'])
const svgRef = ref(null)
const tooltipContainer = ref(null)
const tooltipData = ref(null)
let graphGroup = null
let svg = null

const tooltipStyle = computed(() => {
  if (!tooltipData.value || !tooltipContainer.value) return {}
  const rect = tooltipContainer.value.getBoundingClientRect()
  let x = tooltipData.value.mouseX - rect.left
  let y = tooltipData.value.mouseY - rect.top

  // Approximate tooltip dimensions
  const tooltipWidth = 200
  const tooltipHeight = 30

  // Adjust to prevent overflow
  if (x + tooltipWidth > rect.width) {
    x = rect.width - tooltipWidth - 10
  }
  if (y - tooltipHeight < 0) {
    y = tooltipHeight + 10
  } else {
    y -= tooltipHeight + 10 // Position above mouse
  }

  // Center horizontally
  x -= tooltipWidth / 2

  return {
    left: `${Math.max(0, x)}px`,
    top: `${y}px`,
    transform: 'translateX(-50%)'
  }
})

watch(() => props.nodes, async (nodes) => {
  if (nodes && nodes.length > 0) {
    await nextTick()
    renderChart(nodes)
  }
}, { immediate: true })

const renderChart = (nodes) => {
  d3.select(svgRef.value).selectAll('*').remove()

  svg = d3.select(svgRef.value)
    .attr('width', 800)
    .attr('height', 450)
    .style('background', 'white')
    .call(d3.zoom().on('zoom', zoomed))

  graphGroup = svg.append('g')

  // Compute position counts
  const positionCounts = {}
  nodes.forEach(node => {
    positionCounts[node.position] = (positionCounts[node.position] || 0) + 1
  })
  const data = Object.entries(positionCounts).map(([position, count]) => ({ position, count }))

  if (data.length === 0) return

  const colorScale = d3.scaleOrdinal(d3.schemeCategory10)

  const pie = d3.pie().value(d => d.count)
  const arc = d3.arc().innerRadius(0).outerRadius(120)

  const arcs = pie(data)

  const pieG = graphGroup.append('g')
    .attr('transform', `translate(300, 200)`)

  // Pie slices
  const slices = pieG.selectAll('path')
    .data(arcs)
    .join('path')
    .attr('d', arc)
    .attr('fill', d => colorScale(d.data.position))
    .attr('stroke', 'white')
    .attr('stroke-width', 2)
    .on('mouseover', function(event, d) {
      d3.select(this).transition().duration(200).attr('opacity', 0.8)
      tooltipData.value = {
        text: `${d.data.position}: ${d.data.count} (${((d.data.count / nodes.length) * 100).toFixed(1)}%)`,
        mouseX: event.clientX,
        mouseY: event.clientY
      }
    })
    .on('mouseout', function(event, d) {
      d3.select(this).transition().duration(200).attr('opacity', 1)
      tooltipData.value = null
    })
    .on('click', (event, d) => {
      if (props.onSelect) {
        props.onSelect(d.data.position)
      } else {
        emit('select', d.data.position)
      }
    })

  // Legend positioned to the right, vertically
  const legend = graphGroup.append('g')
    .attr('transform', `translate(500, 100)`)

  legend.selectAll('rect')
    .data(data)
    .join('rect')
    .attr('x', 0)
    .attr('y', (d, i) => i * 24)
    .attr('width', 12)
    .attr('height', 12)
    .attr('fill', d => colorScale(d.position))

  legend.selectAll('text')
    .data(data)
    .join('text')
    .attr('x', 20)
    .attr('y', (d, i) => i * 24 + 12)
    .text(d => `${d.position} (${d.count})`)
    .attr('font-size', 11)
    .attr('fill', '#212529')
    .style('cursor', 'pointer')
    .on('mouseover', function(event, d) {
      tooltipData.value = {
        text: `${d.position}: ${d.count} (${((d.count / nodes.length) * 100).toFixed(1)}%)`,
        mouseX: event.clientX,
        mouseY: event.clientY
      }
    })
    .on('mouseout', () => {
      tooltipData.value = null
    })
    .on('click', (event, d) => {
      if (props.onSelect) {
        props.onSelect(d.position)
      } else {
        emit('select', d.position)
      }
    })
}

function zoomed(event) {
  if (graphGroup) {
    graphGroup.attr('transform', event.transform)
  }
}
</script>