<template>
  <div class="min-h-screen bg-white">
    <header class="bg-gov-blue text-white py-4 sm:py-6">
      <div class="max-w-7xl mx-auto px-4 text-center">
        <h1 class="text-2xl sm:text-3xl font-bold mb-2">Nepo Family Mapper</h1>
        <p class="text-blue-100 text-sm sm:text-base max-w-2xl mx-auto">
          Visualize family ties in Philippine government appointments. Search, zoom, and explore dynasties.
        </p>
      </div>
    </header>
    <main class="max-w-7xl mx-auto px-4 py-6 sm:py-8">
      <div class="grid grid-cols-1 md:grid-cols-2 gap-4 sm:gap-6">
        <div class="bg-white rounded-lg shadow-md p-4 sm:p-6 border border-gray-200 order-2 md:order-1">
          <div class="flex flex-col gap-3 sm:gap-4 mb-4 sm:mb-6">
            <input 
              v-model="searchQuery" 
              type="text" 
              placeholder="Search by name, position, family, or location (e.g., 'Marcos' or 'La Union')"
              class="border-2 border-gov-blue rounded-md px-3 py-2 sm:px-4 sm:py-3 focus:outline-none focus:ring-2 focus:ring-gov-blue focus:border-transparent w-full text-base"
            >
          </div>
          <div class="overflow-auto">
            <NepoGraph 
              v-if="filteredData && filteredData.nodes.length > 0"
              :data="filteredData" 
              :full-node-count="fullNodeCount"
              @node-selected="handleNodeSelected" 
            />
            <div v-else class="flex flex-col items-center justify-center h-96 md:h-[600px] text-center p-4">
              <p class="text-gray-500 text-base sm:text-lg mb-2">No results found</p>
              <p class="text-gray-400 text-sm mb-4">Try adjusting your search or filters.</p>
              <button 
                @click="clearFilters" 
                class="px-4 py-2 bg-gov-blue text-white rounded-md hover:bg-blue-800 transition text-sm"
              >
                Clear Filters
              </button>
            </div>
          </div>
        </div>
        <div class="bg-white rounded-lg shadow-md p-4 sm:p-6 border border-gray-200 order-1 md:order-2">
          <h2 class="text-lg sm:text-xl font-bold text-gray-900 mb-3 sm:mb-4 border-b border-gov-blue pb-2">Details</h2>
          <div v-if="selectedNode" class="space-y-3 sm:space-y-4">
            <button 
              @click="selectedNode = null; connections = []" 
              class="mb-4 inline-flex items-center gap-2 px-3 py-2 bg-gov-blue text-white rounded-md text-sm hover:bg-blue-800 transition"
            >
              ← Back
            </button>
            <div class="bg-gray-50 p-2 sm:p-3 rounded-md">
              <h3 class="font-bold text-base sm:text-lg text-gray-900">{{ selectedNode.name }}</h3>
              <p class="text-sm text-gray-600">{{ selectedNode.position }} ({{ selectedNode.location }})</p>
              <p class="text-xs sm:text-sm text-gray-500">Family: {{ selectedNode.family }}</p>
            </div>
            <div>
              <h4 class="font-semibold text-gray-900 mb-2 text-sm sm:text-base">Connections ({{ connections.length }})</h4>
              <ul class="space-y-2 max-h-32 sm:max-h-40 overflow-y-auto">
                <li v-for="conn in sortedConnections" :key="conn.id" class="text-base sm:text-sm text-gray-700 flex flex-col sm:flex-row sm:justify-between bg-gray-50 p-2 rounded-md gap-1 sm:gap-0">
                  <span class="font-medium">{{ conn.name }}</span>
                  <span class="text-xs sm:text-sm text-gray-500">{{ conn.position }} ({{ conn.location }})</span>
                  <span class="text-xs sm:text-sm text-gov-blue font-medium">{{ conn.relation }}</span>
                </li>
              </ul>
            </div>
          </div>
          <div v-else class="space-y-3 sm:space-y-4">
            <div class="flex flex-col sm:flex-row gap-2 mb-3">
              <h3 class="font-semibold text-gray-900 text-sm sm:text-base border-b border-gov-blue pb-2 flex-1">All Families</h3>
              <div class="flex flex-col gap-2">
                <select v-model="sortOption" class="border border-gray-300 rounded-md px-2 py-1 text-sm focus:outline-none focus:ring-2 focus:ring-gov-blue">
                  <option value="size-desc">Sort: Size (Desc)</option>
                  <option value="name-asc">Sort: Name (Asc)</option>
                  <option value="name-desc">Sort: Name (Desc)</option>
                </select>
                
              </div>
            </div>
            <ul v-if="sortedFamilies.length > 0" class="space-y-2">
              <li v-for="family in sortedFamilies" :key="family.name" class="text-base sm:text-sm text-gray-700 flex flex-col sm:flex-row sm:justify-between cursor-pointer hover:bg-gov-blue hover:bg-opacity-10 p-3 rounded-md border border-gray-200 gap-1 sm:gap-2" @click="filterByFamily(family.name)">
                <span>{{ family.name }}</span>
                <span class="text-xs sm:text-sm text-gray-500">{{ family.count }} members</span>
              </li>
            </ul>
            <div v-else class="text-center py-8">
              <p class="text-gray-500 text-base sm:text-lg mb-2">No families found</p>
              <p class="text-gray-400 text-sm mb-4">Adjust your sorting or filters to see results.</p>
              <button 
                @click="clearFilters" 
                class="px-4 py-2 bg-gov-blue text-white rounded-md hover:bg-blue-800 transition text-sm"
              >
                Clear Filters
              </button>
            </div>
          </div>
        </div>
      </div>
    </main>
    <footer class="bg-gray-800 text-gray-300 py-4 sm:py-6 mt-8 sm:mt-12">
      <div class="max-w-7xl mx-auto px-4 text-center">
        <p class="text-sm sm:text-base">Data from 2025 public disclosures</p>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import * as d3 from 'd3'
import NepoGraph from './components/NepoGraph.vue'

const graphData = ref(null)
const searchQuery = ref('')
const selectedNode = ref(null)
const connections = ref([])
const currentData = ref(null)
const sortOption = ref('size-desc')

onMounted(async () => {
  graphData.value = await d3.json('/data.json')
  currentData.value = graphData.value
})

const fullNodeCount = computed(() => graphData.value?.nodes?.length || 0)

const uniquePositions = computed(() => {
  if (!graphData.value) return []
  return [...new Set(graphData.value.nodes.map(node => node.position))].sort()
})

const uniqueLocations = computed(() => {
  if (!graphData.value) return []
  return [...new Set(graphData.value.nodes.map(node => node.location))].sort()
})

const filteredData = computed(() => {
  if (!graphData.value) return null
  let resultNodes = graphData.value.nodes
  let resultLinks = graphData.value.links

  // Search filter
  if (searchQuery.value.trim()) {
    const query = searchQuery.value.toLowerCase()
    resultNodes = resultNodes.filter(node => 
      node.name.toLowerCase().includes(query) ||
      node.position.toLowerCase().includes(query) ||
      node.family.toLowerCase().includes(query) ||
      node.location.toLowerCase().includes(query)
    )
  }

  const nodeIds = new Set(resultNodes.map(n => n.id))
  resultLinks = graphData.value.links.filter(link => 
    nodeIds.has(link.source) || nodeIds.has(link.target)
  )

  const result = { nodes: resultNodes, links: resultLinks }
  currentData.value = result
  return result
})

const sortedFamilies = computed(() => {
  if (!graphData.value) return []
  const familyMap = {}
  graphData.value.nodes.forEach(node => {
    familyMap[node.family] = (familyMap[node.family] || 0) + 1
  })
  let families = Object.entries(familyMap)
    .map(([name, count]) => ({ name, count }))
  switch (sortOption.value) {
    case 'size-desc':
      families.sort((a, b) => b.count - a.count)
      break
    case 'name-asc':
      families.sort((a, b) => a.name.localeCompare(b.name))
      break
    case 'name-desc':
      families.sort((a, b) => b.name.localeCompare(a.name))
      break
  }
  return families
})

const sortedConnections = computed(() => {
  return [...connections.value].sort((a, b) => a.position.localeCompare(b.position))
})

const handleNodeSelected = (node) => {
  selectedNode.value = node
  if (currentData.value) {
    connections.value = currentData.value.links
      .filter(link => link.source === node.id || link.target === node.id)
      .map(link => {
        const otherId = link.source === node.id ? link.target : link.source
        const otherNode = currentData.value.nodes.find(n => n.id === otherId)
        return {
          id: otherId,
          name: otherNode ? otherNode.name : 'Unknown',
          position: otherNode ? otherNode.position : '',
          location: otherNode ? otherNode.location : '',
          relation: link.relation
        }
      })
  }
}

const filterByFamily = (familyName) => {
  searchQuery.value = familyName
}

const clearFilters = () => {
  searchQuery.value = ''
  sortOption.value = 'size-desc'
}
</script>