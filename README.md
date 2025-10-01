# Nepo Family Mapper

A interactive visualization tool to map political family ties (nepotism) in Philippine government, powered by open data. Explore dynasties like Marcos, Duterte, and more through a force-directed graph. Contribute via Google Sheets for easy data expansion.

## Tech Stack
Frontend: Vue.js (Vue 3, Composition API)  
Styling: Tailwind CSS  
Backend: None (static CSV fetch from Google Sheets)  
Database: Google Sheets (as lightweight backend)  
Deployment: Vercel  

## Key Features
- Interactive force-directed graph with zoom/pan, node dragging, and highlights.
- Real-time search/filter by name, family, position, or location.
- Dropdown sorting for families (by size/name) and filters for position/location.
- "No Results Found" message with clear filters button.
- Node details with sorted connections list and back button.
- Label toggle for less clutter.
- Data expandable via Google Sheets (CSV fetch with PapaParse).

## Content Aggregation
Sources: Google Sheets (Nodes tab: officials; Links tab: family ties) or fallback `data.json` (35+ nodes from 2025 disclosures).  
Logic: Fetch/parse CSVs on load; extract unique positions/locations for dropdowns. Rank families by member count. Cache in-memory; refresh on Sheet edits (reload app). Verified from public sources (COMELEC, Congress API, news like Rappler).

## Sample Code
### Frontend (App.vue)
```vue
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
            <label class="flex items-center gap-2 text-sm sm:text-base text-gray-700">
              <input type="checkbox" v-model="showLabels" class="rounded border-gov-blue w-4 h-4 sm:w-5 sm:h-5">
              <span>Show Labels</span>
            </label>
          </div>
          <div class="overflow-auto">
            <NepoGraph 
              v-if="filteredData && filteredData.nodes.length > 0"
              :data="filteredData" 
              :show-labels="showLabels"
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
        <!-- Details panel omitted for brevity -->
      </div>
    </main>
    <footer class="bg-gray-800 text-gray-300 py-4 sm:py-6 mt-8 sm:mt-12">
      <div class="max-w-7xl mx-auto px-4 text-center">
        <p class="text-sm sm:text-base">Open-source for BetterGov.ph | Data from 2025 public disclosures</p>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import * as d3 from 'd3'
import Papa from 'papaparse'
import NepoGraph from './components/NepoGraph.vue'

// Refs and state...
const graphData = ref(null)
// ... (full script as in previous response)
</script>
```

### Data Fetch (/api/data.js – Optional Vercel Function for Proxy)
```javascript
export default async function handler(req, res) {
  try {
    const nodesRes = await fetch(process.env.NODES_CSV_URL);
    const nodesCsv = await nodesRes.text();
    const nodes = Papa.parse(nodesCsv, { header: true }).data;

    const linksRes = await fetch(process.env.LINKS_CSV_URL);
    const linksCsv = await linksRes.text();
    const links = Papa.parse(linksCsv, { header: true }).data;

    res.status(200).json({ nodes, links });
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch data' });
  }
}

```
How to Contribute
We welcome contributions to expand data, fix bugs, or improve features! This project is part of BetterGov.ph's civic tech ecosystem – help fight nepotism by adding verified family ties.

1. Contribute Data (Easiest – No Code Needed)

Edit our shared Google Sheet: Nepo Mapper Data Sheet (Anyone with link can edit).

Nodes Tab: Add a row for a new official (e.g., name: "New Official", position: "Mayor, Example", family: "Example Clan", location: "Example Province"). ID auto-fills.
Links Tab: Connect IDs (e.g., source: 1, target: 37, relation: "Father-Daughter").

Always cite sources (add a "source" column or comment in Sheet).
Changes live after app reload (5-10 mins for Sheet sync).

2. Suggest Data or Ideas

Open a GitHub Issue: Use the template for "New Family Suggestion" (include name, sources, e.g., COMELEC link).
Example: "Add Padilla Dynasty: Robin Padilla (Senator), Sources: [Link]".

3. Code Contributions

Fork the repo.
Create a branch: git checkout -b feature/add-filter.
Commit changes: git commit -m "Add advanced search filter".
Push & PR: Reference any related Issue.
Run locally: npm install && npm run dev.

4. Guidelines

Data: Verified facts only (no rumors); 2+ family members per dynasty.
Code: Follow Vue 3 style; add tests if possible (Vitest).
Ethics: For advocacy – respect privacy, focus on public roles.
Questions? Join BetterGov.ph Discord or open an Issue.

Thanks for contributing – together, we're building transparency!
Challenges
Sheet Sync: Changes require app reload; add polling for live updates.
CSV Parsing: Handle edge cases like missing headers (PapaParse robust).
Contributions: Moderate edits to avoid spam; use Google Forms for submissions.
Scalability: Sheets for <500 nodes; migrate to Supabase for 1k+.

## Challenges
Sheet Sync: Changes require app reload; add polling for live updates.  
CSV Parsing: Handle edge cases like missing headers (PapaParse robust).  
Contributions: Moderate edits to avoid spam; use Google Forms for submissions.  
Scalability: Sheets for <500 nodes; migrate to Supabase for 1k+.

## Timeline
Total: 4 days (setup + integrate + test).  

## Next Steps
- Fork this repo and deploy to Vercel.  
- Create Google Sheet → Publish CSVs → Update URLs in code.  
- Test: Add a new node in Sheet → Reload → See in graph.  
- Promote: Share on X/BetterGov.ph Discord for contributions!