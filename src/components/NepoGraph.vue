<template>
  <svg ref="svgRef" class="w-full h-96 md:h-[600px]"></svg>
</template>

<script setup>
import { ref, watch, nextTick } from 'vue'
import * as d3 from 'd3'

const props = defineProps({
  data: { type: Object, default: null },
  showLabels: { type: Boolean, default: true },
  fullNodeCount: { type: Number, default: 0 }
})

const emit = defineEmits(['node-selected'])
const svgRef = ref(null)
let simulation = null
let graphGroup = null
let linkSel = null
let nodeSel = null
let labelSel = null
const width = 800
const height = 600

watch(() => props.data, async (newData) => {
  if (newData) {
    await nextTick()
    d3.select(svgRef.value).selectAll('*').remove()
    renderGraph(newData)
    if (newData.nodes.length > 0 && newData.nodes.length < props.fullNodeCount) {
      setTimeout(() => {
        zoomToFit(newData.nodes)
      }, 800)
    }
  }
}, { immediate: true })

watch(() => props.showLabels, () => {
  if (labelSel) {
    labelSel.attr('display', props.showLabels ? 'block' : 'none')
  }
})

const renderGraph = (data) => {
  if (!data) return

  const svg = d3.select(svgRef.value)
    .attr('width', width)
    .attr('height', height)
    .style('background', 'white')
    .call(d3.zoom().on('zoom', zoomed))
    .on('click', resetHighlight)

  graphGroup = svg.append('g')

  // Prepare links: Only those with both ends in data.nodes
  const links = data.links
    .filter(l => data.nodes.find(n => n.id === l.source) && data.nodes.find(n => n.id === l.target))
    .map(l => ({ 
      ...l, 
      source: data.nodes.find(n => n.id === l.source), 
      target: data.nodes.find(n => n.id === l.target) 
    }))

  simulation = d3.forceSimulation(data.nodes)
    .force('link', d3.forceLink(links).id(d => d.id).distance(120))
    .force('charge', d3.forceManyBody().strength(-300))
    .force('center', d3.forceCenter(width / 2, height / 2))

  linkSel = graphGroup.append('g')
    .selectAll('line')
    .data(links)
    .join('line')
    .attr('stroke', '#0066eb')
    .attr('stroke-opacity', 0.6)
    .attr('stroke-width', 2)

  nodeSel = graphGroup.append('g')
    .selectAll('circle')
    .data(data.nodes)
    .join('circle')
    .attr('r', 8)
    .attr('fill', d => d.type === 'official' ? '#0066eb' : '#10b981')
    .attr('stroke', '#ffffff')
    .attr('stroke-width', 2)
    .call(d3.drag()
      .on('start', dragstarted)
      .on('drag', dragged)
      .on('end', dragended)
    )
    .on('click', (event, d) => {
      event.stopPropagation()
      emit('node-selected', d)
      highlightConnections(d, data)
    })
    .on('mouseover', (event, d) => {
      d3.select(event.currentTarget).transition().duration(200).attr('r', 12).attr('fill', '#0066eb')
    })
    .on('mouseout', (event, d) => {
      d3.select(event.currentTarget).transition().duration(200).attr('r', 8).attr('fill', d.type === 'official' ? '#0066eb' : '#10b981')
    })

  labelSel = graphGroup.append('g')
    .selectAll('text')
    .data(data.nodes)
    .join('text')
    .text(d => d.name)
    .attr('dx', 12)
    .attr('dy', '.35em')
    .attr('font-size', 9)
    .attr('fill', '#212529')
    .attr('display', props.showLabels ? 'block' : 'none')

  simulation.on('tick', ticked)

  function ticked() {
    linkSel
      .attr('x1', d => d.source.x)
      .attr('y1', d => d.source.y)
      .attr('x2', d => d.target.x)
      .attr('y2', d => d.target.y)

    nodeSel
      .attr('cx', d => d.x)
      .attr('cy', d => d.y)

    labelSel
      .attr('x', d => d.x)
      .attr('y', d => d.y)
  }
}

function zoomed(event) {
  graphGroup.attr('transform', event.transform)
}

function zoomToFit(nodes) {
  if (!graphGroup || nodes.length === 0) return
  const padding = 40
  const minX = d3.min(nodes, d => d.x)
  const maxX = d3.max(nodes, d => d.x)
  const minY = d3.min(nodes, d => d.y)
  const maxY = d3.max(nodes, d => d.y)
  const dx = maxX - minX
  const dy = maxY - minY
  const x = (minX + maxX) / 2
  const y = (minY + maxY) / 2
  const scale = 0.8 / Math.max(dx / width, dy / height, 0.1)
  const translate = [width / 2 - scale * x, height / 2 - scale * y]
  graphGroup.transition().duration(750)
    .attr('transform', `translate(${translate}) scale(${scale})`)
}

function resetHighlight() {
  if (linkSel) linkSel.transition().duration(200).style('stroke-opacity', 0.6).style('stroke-width', 2)
  if (nodeSel) nodeSel.transition().duration(200).attr('r', 8).attr('fill', d => d.type === 'official' ? '#0066eb' : '#10b981')
}

function dragstarted(event, d) {
  if (!event.active) simulation.alphaTarget(0.3).restart()
  d.fx = d.x
  d.fy = d.y
}

function dragged(event, d) {
  d.fx = event.x
  d.fy = event.y
}

function dragended(event, d) {
  if (!event.active) simulation.alphaTarget(0)
  d.fx = null
  d.fy = null
}

function highlightConnections(selectedNode, data) {
  if (!linkSel || !nodeSel) return
  linkSel.transition().duration(200)
    .style('stroke-opacity', d => (d.source.id === selectedNode.id || d.target.id === selectedNode.id) ? 1 : 0.1)
    .style('stroke-width', d => (d.source.id === selectedNode.id || d.target.id === selectedNode.id) ? 4 : 2)

  nodeSel.transition().duration(200)
    .attr('r', d => (d.id === selectedNode.id) ? 15 : 8)
    .attr('fill', d => {
      if (d.id === selectedNode.id) return '#ef4444'
      // Fixed: Use raw links from data (IDs are numbers)
      const isConnected = data.links.some(l => (l.source === selectedNode.id && l.target === d.id) || (l.source === d.id && l.target === selectedNode.id))
      return isConnected ? '#f59e0b' : (d.type === 'official' ? '#0066eb' : '#10b981')
    })
}
</script>