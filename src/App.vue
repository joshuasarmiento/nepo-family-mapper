<template>
  <div class="min-h-screen bg-white">
    <header class="bg-gov-blue text-white py-4 sm:py-6">
      <div class="max-w-7xl mx-auto px-4 text-center">
        <h1 class="text-2xl sm:text-3xl font-bold mb-2">Political Dynasties in the Philippines</h1>
        <p class="text-blue-100 text-sm sm:text-base max-w-2xl mx-auto">
          Visualize family ties in Philippine government appointments. Search, zoom, and explore dynasties.
        </p>
      </div>
      <!-- <div class="max-w-6xl mx-auto px-4 py-6 sm:py-8">
        <div class="grid grid-cols-1 md:grid-cols-2 gap-4 sm:gap-6">
          <input v-model="searchQuery" type="text"
            placeholder="Search by name, position, family, or location (e.g., 'Marcos' or 'La Union')"
            class="border-2 border-gov-blue rounded-md px-3 py-2 sm:px-4 sm:py-3 focus:border-transparent w-full text-base">
          <div class="flex flex-col sm:flex-row gap-2">
            <select v-model="sortOption"
              class="border border-gray-300 rounded-md px-2 py-1 text-sm">
              <option value="size-desc" class="text-gray-900">Sort: Size (Desc)</option>
              <option value="name-asc" class="text-gray-900">Sort: Name (Asc)</option>
              <option value="name-desc" class="text-gray-900">Sort: Name (Desc)</option>
            </select>
            <select v-model="selectedPosition"
              class="border border-gray-300 rounded-md px-2 py-1 text-sm">
              <option value="" class="text-gray-900">Position: All</option>
              <option v-for="pos in uniquePositions" :key="pos" :value="pos" class="text-gray-900">{{ pos }}</option>
            </select>
            <select v-model="selectedLocation"
              class="border border-gray-300 rounded-md px-2 py-1 text-sm">
              <option value="" class="text-gray-900">Location: All</option>
              <option v-for="loc in uniqueLocations" :key="loc" :value="loc" class="text-gray-900">{{ loc }}</option>
            </select>
          </div>
        </div>
      </div> -->
    </header>
    <main class="max-w-7xl mx-auto px-4 py-6 sm:py-8">
      <div class="grid grid-cols-1 md:grid-cols-2 gap-4 sm:gap-6">
        <div class="p-4 sm:p-6 order-2 md:order-1">
          <div class="flex flex-col gap-3 sm:gap-4 mb-4 sm:mb-6">
            <label class="flex items-center gap-2 text-sm sm:text-base text-gray-700">
              <input type="checkbox" v-model="showLabels" class="rounded border-gov-blue w-4 h-4 sm:w-5 sm:h-5">
              <span>Show Labels</span>
            </label>
          </div>
          <div class="border border-gray-200 rounded-md overflow-hidden">
            <!-- Tabs -->
            <div class="flex bg-gray-100">
              <button
                @click="activeTab = 'network'"
                :class="[
                  'px-4 py-2 text-sm font-medium',
                  activeTab === 'network'
                    ? 'bg-white text-gov-blue border-b-2 border-gov-blue'
                    : 'text-gray-500 hover:text-gray-700 hover:bg-gray-50'
                ]"
              >
                Network Graph
              </button>
              <button
                @click="activeTab = 'bar'"
                :class="[
                  'px-4 py-2 text-sm font-medium',
                  activeTab === 'bar'
                    ? 'bg-white text-gov-blue border-b-2 border-gov-blue'
                    : 'text-gray-500 hover:text-gray-700 hover:bg-gray-50'
                ]"
              >
                Family Sizes
              </button>
              <button
                @click="activeTab = 'pie'"
                :class="[
                  'px-4 py-2 text-sm font-medium',
                  activeTab === 'pie'
                    ? 'bg-white text-gov-blue border-b-2 border-gov-blue'
                    : 'text-gray-500 hover:text-gray-700 hover:bg-gray-50'
                ]"
              >
                Position Distribution
              </button>
            </div>
            <!-- Tab Content -->
            <div class="overflow-auto h-[600px] bg-white">
              <!-- Network Graph Tab -->
              <div v-if="activeTab === 'network'">
                <NepoGraph v-if="filteredData && filteredData.nodes.length > 0" :data="filteredData"
                  :show-labels="showLabels" :full-node-count="fullNodeCount" @node-selected="handleNodeSelected" />
                <div v-else class="flex flex-col items-center justify-center h-full text-center p-4">
                  <p class="text-gray-500 text-base sm:text-lg mb-2">No results found</p>
                  <p class="text-gray-400 text-sm mb-4">Try adjusting your search or filters.</p>
                  <button @click="clearFilters"
                    class="px-4 py-2 bg-gov-blue text-white rounded-md hover:bg-blue-800 transition text-sm">
                    Clear Filters
                  </button>
                </div>
              </div>
              <!-- Bar Chart Tab -->
              <div v-else-if="activeTab === 'bar'" class="flex items-center justify-center h-full">
                <FamilyBarChart v-if="sortedFamilies.length > 0" :families="sortedFamilies" @select="filterByFamily" />
                <div v-else class="flex flex-col items-center justify-center text-center p-4">
                  <p class="text-gray-500 text-base sm:text-lg mb-2">No families found</p>
                  <p class="text-gray-400 text-sm mb-4">Adjust your sorting or filters to see results.</p>
                  <button @click="clearFilters"
                    class="px-4 py-2 bg-gov-blue text-white rounded-md hover:bg-blue-800 transition text-sm">
                    Clear Filters
                  </button>
                </div>
              </div>
              <!-- Pie Chart Tab -->
              <div v-else-if="activeTab === 'pie'" class="flex items-center justify-center h-full">
                <PositionPieChart v-if="preFamilyFilteredNodes.length > 0" :nodes="preFamilyFilteredNodes" @select="selectPosition" />
                <div v-else class="flex flex-col items-center justify-center text-center p-4">
                  <p class="text-gray-500 text-base sm:text-lg mb-2">No data available</p>
                  <p class="text-gray-400 text-sm mb-4">Adjust your filters to see results.</p>
                  <button @click="clearFilters"
                    class="px-4 py-2 bg-gov-blue text-white rounded-md hover:bg-blue-800 transition text-sm">
                    Clear Filters
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div class="bg-white rounded-lg shadow-md p-4 sm:p-6 border border-gray-200 order-1 md:order-2">
          <div v-if="selectedNode" class="space-y-3 sm:space-y-4">
            <button @click="handleBackClick"
              class="mb-4 inline-flex items-center gap-2 px-3 py-2 bg-gov-blue text-white rounded-md text-sm hover:bg-blue-800 transition">
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
                <li v-for="conn in sortedConnections" :key="conn.id"
                  class="text-base sm:text-sm text-gray-700 flex flex-col sm:flex-row sm:justify-between bg-gray-50 p-2 rounded-md gap-1 sm:gap-0">
                  <span class="font-medium">{{ conn.name }}</span>
                  <span class="text-xs sm:text-sm text-gray-500">{{ conn.position }} ({{ conn.location }})</span>
                  <span class="text-xs sm:text-sm text-gov-blue font-medium">{{ conn.relation }}</span>
                </li>
              </ul>
            </div>
          </div>
          <div v-else-if="selectedFamily" class="space-y-3 sm:space-y-4">
            <button @click="handleBackClick"
              class="mb-4 inline-flex items-center gap-2 px-3 py-2 bg-gov-blue text-white rounded-md text-sm hover:bg-blue-800 transition">
              ← Back
            </button>
            <div class="bg-gray-50 p-2 sm:p-3 rounded-md">
              <h3 class="font-bold text-base sm:text-lg text-gray-900">{{ selectedFamily }}</h3>
              <p class="text-sm text-gray-600">{{ filteredData.nodes.length }} members</p>
            </div>
            <div>
              <h4 class="font-semibold text-gray-900 mb-2 text-sm sm:text-base">Family Members ({{ filteredData.nodes.length }})</h4>
              <ul class="space-y-2 max-h-32 sm:max-h-40 overflow-y-auto">
                <li v-for="member in filteredData.nodes" :key="member.id"
                  class="text-base sm:text-sm text-gray-700 flex flex-col sm:flex-row sm:justify-between cursor-pointer hover:bg-gov-blue hover:bg-opacity-10 bg-gray-50 p-2 rounded-md gap-1 sm:gap-0"
                  @click="handleNodeSelected(member)">
                  <span class="font-medium">{{ member.name }}</span>
                  <span class="text-xs sm:text-sm text-gray-500">{{ member.position }} ({{ member.location }})</span>
                  <span class="text-xs sm:text-sm text-gov-blue font-medium">Family Member</span>
                </li>
              </ul>
            </div>
          </div>
          <div v-else class="space-y-3 sm:space-y-4">
            <div class="flex flex-col sm:flex-row gap-2 mb-3">
            </div>
            <ul v-if="sortedFamilies.length > 0" class="space-y-2">
              <li v-for="family in sortedFamilies" :key="family.name"
                class="text-base sm:text-sm text-gray-700 flex flex-col sm:flex-row sm:justify-between cursor-pointer hover:bg-gov-blue hover:bg-opacity-10 p-3 rounded-md border border-gray-200 gap-1 sm:gap-2"
                @click="filterByFamily(family.name)">
                <span>{{ family.name }}</span>
                <span class="text-xs sm:text-sm text-gray-500">{{ family.count }} members</span>
              </li>
            </ul>
            <div v-else class="text-center py-8">
              <p class="text-gray-500 text-base sm:text-lg mb-2">No families found</p>
              <p class="text-gray-400 text-sm mb-4">Adjust your sorting or filters to see results.</p>
              <button @click="clearFilters"
                class="px-4 py-2 bg-gov-blue text-white rounded-md hover:bg-blue-800 transition text-sm">
                Clear Filters
              </button>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import * as d3 from 'd3'
import Papa from 'papaparse'
import NepoGraph from './components/NepoGraph.vue'
import FamilyBarChart from './components/FamilyBarChart.vue'
import PositionPieChart from './components/PositionPieChart.vue'

const graphData = ref(null)
const searchQuery = ref('')
const showLabels = ref(true)
const selectedNode = ref(null)
const selectedFamily = ref(null)
const connections = ref([])
const sortOption = ref('size-desc')
const selectedPosition = ref('')
const selectedLocation = ref('')
const activeTab = ref('network')

// Replace with your published CSV URLs
const NODES_CSV_URL = 'https://docs.google.com/spreadsheets/d/e/2PACX-1vT5qa3Oqb42A6imaxGDXHu5k6qQ8hZSJF-LveKFsdRwkoda_qlltng6Kbp-psoJgyh1p-z7Ziw8V1SZ/pub?gid=0&single=true&output=csv'
const LINKS_CSV_URL = 'https://docs.google.com/spreadsheets/d/e/2PACX-1vT5qa3Oqb42A6imaxGDXHu5k6qQ8hZSJF-LveKFsdRwkoda_qlltng6Kbp-psoJgyh1p-z7Ziw8V1SZ/pub?gid=1481152180&single=true&output=csv'

const loadData = async () => {
  try {
    // Fetch Nodes CSV
    const nodesRes = await fetch(NODES_CSV_URL)
    const nodesCsv = await nodesRes.text()
    const nodesData = Papa.parse(nodesCsv, {
      header: true,
      skipEmptyLines: true,
      dynamicTyping: { id: true }
    }).data

    // Fetch Links CSV
    const linksRes = await fetch(LINKS_CSV_URL)
    const linksCsv = await linksRes.text()
    const linksData = Papa.parse(linksCsv, {
      header: true,
      skipEmptyLines: true,
      dynamicTyping: { source: true, target: true }
    }).data

    graphData.value = {
      nodes: nodesData.filter(row => row.name),  // Skip empty rows
      links: linksData.filter(row => row.relation)  // Skip empty links
    }
  } catch (error) {
    console.error('Failed to load data from Sheet:', error)
    // Fallback to local JSON
    graphData.value = await d3.json('/data.json')
  }
}

onMounted(loadData)

const fullNodeCount = computed(() => graphData.value?.nodes?.length || 0)

const uniquePositions = computed(() => {
  if (!graphData.value) return []
  return [...new Set(graphData.value.nodes.map(node => node.position))].sort()
})

const uniqueLocations = computed(() => {
  if (!graphData.value) return []
  return [...new Set(graphData.value.nodes.map(node => node.location))].sort()
})

const preFamilyFilteredNodes = computed(() => {
  if (!graphData.value?.nodes) return []
  return graphData.value.nodes.filter(node => {
    // Search filter
    if (searchQuery.value.trim()) {
      const query = searchQuery.value.toLowerCase()
      if (
        !node.name.toLowerCase().includes(query) &&
        !node.position.toLowerCase().includes(query) &&
        !node.family.toLowerCase().includes(query) &&
        !node.location.toLowerCase().includes(query)
      ) {
        return false
      }
    }
    // Position filter
    if (selectedPosition.value && node.position !== selectedPosition.value) {
      return false
    }
    // Location filter
    if (selectedLocation.value && node.location !== selectedLocation.value) {
      return false
    }
    return true
  })
})

const filteredData = computed(() => {
  if (!graphData.value) return { nodes: [], links: [] }
  let nodes = preFamilyFilteredNodes.value
  if (selectedFamily.value) {
    nodes = nodes.filter(node => node.family === selectedFamily.value)
  }
  const nodeIds = new Set(nodes.map(n => n.id))
  const links = graphData.value.links.filter(link =>
    nodeIds.has(link.source) && nodeIds.has(link.target)
  )
  return { nodes, links }
})

const sortedFamilies = computed(() => {
  if (!preFamilyFilteredNodes.value.length) return []
  const familyMap = {}
  preFamilyFilteredNodes.value.forEach(node => {
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
  selectedFamily.value = null
  if (graphData.value) {
    connections.value = graphData.value.links
      .filter(link => link.source === node.id || link.target === node.id)
      .map(link => {
        const otherId = link.source === node.id ? link.target : link.source
        const otherNode = graphData.value.nodes.find(n => n.id === otherId)
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

const handleBackClick = () => {
  if (selectedNode.value) {
    selectedNode.value = null
  } else if (selectedFamily.value) {
    selectedFamily.value = null
  }
  connections.value = []
}

const filterByFamily = (familyName) => {
  selectedFamily.value = familyName
  selectedNode.value = null
}

const selectPosition = (position) => {
  selectedPosition.value = selectedPosition.value === position ? '' : position
}

const clearFilters = () => {
  searchQuery.value = ''
  selectedPosition.value = ''
  selectedLocation.value = ''
  sortOption.value = 'size-desc'
  selectedFamily.value = null
  selectedNode.value = null
  connections.value = []
}
</script>