<template>
  <div class="relative">
    <svg ref="svgRef" class="w-full h-96 md:h-[600px]"></svg>
    <div class="absolute top-2 right-2 flex flex-col gap-2 z-10">
      <button
        @click="zoomIn"
        type="button"
        class="px-3 py-1 bg-gray-200 text-gray-700 rounded-md text-sm hover:bg-gray-300 focus-visible:ring-2 focus-visible:ring-gov-blue focus-visible:ring-offset-2 transition-colors outline-none"
        :aria-label="`Zoom in on the graph`"
        title="Zoom in (+)"
      >
        +
      </button>
      <button
        @click="zoomOut"
        type="button"
        class="px-3 py-1 bg-gray-200 text-gray-700 rounded-md text-sm hover:bg-gray-300 focus-visible:ring-2 focus-visible:ring-gov-blue focus-visible:ring-offset-2 transition-colors outline-none"
        :aria-label="`Zoom out on the graph`"
        title="Zoom out (-)"
      >
        -
      </button>
      <button
        @click="center"
        type="button"
        class="px-3 py-1 bg-gov-blue text-white rounded-md text-sm hover:bg-blue-800 focus-visible:ring-2 focus-visible:ring-gov-blue focus-visible:ring-offset-2 transition-colors outline-none"
        :aria-label="`Center the graph view`"
        title="Center view"
      >
        Center
      </button>
    </div>
  </div>
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
let svg = null
let zoomBehavior = null
const width = 800
const height = 600
const DEFAULT_ZOOM = 0.4

watch(() => props.data, async (newData) => {
  if (newData) {
    await nextTick()
    d3.select(svgRef.value).selectAll('*').remove()
    renderGraph(newData)
    if (newData.nodes.length > 0 && newData.nodes.length < props.fullNodeCount) {
      setTimeout(() => {
        zoomToFit(newData.nodes)
      }, 800)
    } else {
      // Apply default zoom on full load
      setTimeout(() => {
        const newTransform = d3.zoomIdentity.scale(DEFAULT_ZOOM)
        if (svg && zoomBehavior) {
          svg.transition().duration(750).call(zoomBehavior.transform, newTransform)
        }
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

  svg = d3.select(svgRef.value)
    .attr('width', width)
    .attr('height', height)
    .style('background', 'white')
    .on('click', (event) => {
      event.stopPropagation()
      resetHighlight()
    })

  zoomBehavior = d3.zoom()
    .scaleExtent([0.1, 10])
    .on('zoom', zoomed)
  svg.call(zoomBehavior)

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
  if (!svg || !zoomBehavior || nodes.length === 0) return
  const padding = 40
  const minX = d3.min(nodes, d => d.x)
  const maxX = d3.max(nodes, d => d.x)
  const minY = d3.min(nodes, d => d.y)
  const maxY = d3.max(nodes, d => d.y)
  if (minX === undefined || maxX === undefined || minY === undefined || maxY === undefined) return
  const dx = maxX - minX
  const dy = maxY - minY
  const x = (minX + maxX) / 2
  const y = (minY + maxY) / 2
  const scale = 0.8 / Math.max(dx / width, dy / height, 0.1)
  const translateX = width / 2 - scale * x
  const translateY = height / 2 - scale * y
  const newTransform = d3.zoomIdentity.translate(translateX, translateY).scale(scale)
  svg.transition().duration(750).call(zoomBehavior.transform, newTransform)
}

const zoomIn = () => {
  if (!svg || !zoomBehavior) return
  const current = d3.zoomTransform(svg.node())
  const newTransform = current.scaleBy(1.2)
  svg.transition().duration(750).call(zoomBehavior.transform, newTransform)
}

const zoomOut = () => {
  if (!svg || !zoomBehavior) return
  const current = d3.zoomTransform(svg.node())
  const newTransform = current.scaleBy(0.8)
  svg.transition().duration(750).call(zoomBehavior.transform, newTransform)
}

const center = () => {
  if (props.data && props.data.nodes.length > 0) {
    zoomToFit(props.data.nodes)
  } else if (svg && zoomBehavior) {
    const newTransform = d3.zoomIdentity.scale(DEFAULT_ZOOM)
    svg.transition().duration(750).call(zoomBehavior.transform, newTransform)
  }
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