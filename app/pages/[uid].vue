<template>
  <!-- <div class="menu">
    <ul>
      <li>
        <NuxtLink :to="/france">France</NuxtLink>
      </li>
      <li>
        <NuxtLink :to="/germany">Germany</NuxtLink>
      </li>
    </ul>
  </div> -->
  <div class="diagram">
    <div v-if="prismicData" class="prismic-header">
      <h1>{{ prismicData.title }}</h1>
    </div>
    <div ref="chartContainer" class="chart-container"></div>
    <div v-if="loading" class="loading">Loading data...</div>
    <div v-if="error" class="error">{{ error }}</div>
    <div class="debug-info">
      <p>Debug: {{ debugInfo }}</p>
    </div>
  </div>
</template>

<script setup>
import * as d3 from 'd3'
import { sankey, sankeyLinkHorizontal } from 'd3-sankey'
import { onMounted, ref, onUnmounted } from 'vue'

// ===== PRISMIC INTEGRATION =====
const route = useRoute()
const { client } = usePrismic()

// ===== CONSTANTS & CONFIGURATION =====
const CHART_CONFIG = {
  margins: { top: 20, right: 400, bottom: 20, left: 20 },
  minWidth: 300,
  minHeight: 500,
  nodeWidth: 80,
  nodePadding: 15,
  cornerRadius: 8,
  defaultOpacity: 0.5, // Default opacity for all elements (50%)
  linkOpacityHover: 1, // Full opacity on hover
  backgroundOpacity: 0.1, // 10% opacity when backgrounded
  nodeOpacityHighlight: 0.9, // 90% opacity when highlighted
  fontSize: { base: 10, min: 10, max: 14, title: { min: 14, max: 18 } },
  tooltipOffset: 15
}

const COLORS = {
  custom: ['#8863EB', '#E163EB', '#636AEB', '#EB63AB', '#B463EB', '#D3B1EB'],
  event: '#2E8B57',
  border: '#000',
  text: { primary: '#333', white: '#fff' },
  tooltip: { bg: 'rgba(0,0,0,0.9)', shadow: '0 4px 8px rgba(0,0,0,0.3)' }
}

// ===== REACTIVE STATE =====
const chartContainer = ref(null)
const loading = ref(true)
const error = ref(null)
const debugInfo = ref('Initializing...')
const currentData = ref(null)
const prismicData = ref(null)

// ===== RESIZE OBSERVER =====
let resizeObserver = null

// ===== LIFECYCLE HOOKS =====
onMounted(async () => {
  debugInfo.value = 'Component mounted, starting Prismic data load...'
  await loadPrismicData()
  setupResizeObserver()
})

onUnmounted(() => {
  if (resizeObserver) resizeObserver.disconnect()
})

// ===== PRISMIC DATA LOADING =====
async function loadPrismicData() {
  try {
    loading.value = true
    error.value = null
    debugInfo.value = 'Loading Prismic data...'
    
    // Fetch the diagram page from Prismic
    const { data } = await client.getByUID('diagram', route.params.uid)
    
    if (!data) {
      throw new Error('Diagram page not found')
    }
    
    prismicData.value = data
    debugInfo.value = `Prismic data loaded: ${data.title}`
    
    // Load CSV data from the csv_file field
    if (data.csv_file?.url) {
      await loadCSVData(data.csv_file.url)
    } else {
      throw new Error('CSV file not found in Prismic data')
    }
    
  } catch (err) {
    console.error('Error loading Prismic data:', err)
    error.value = err.message
    debugInfo.value = `Prismic failed: ${err.message}`
    loading.value = false
  }
}

// ===== CSV DATA LOADING =====
async function loadCSVData(csvUrl) {
  try {
    debugInfo.value = 'Loading CSV data from Prismic...'
    
    const csvData = await d3.csv(csvUrl)
    console.log('Raw CSV data from Prismic:', csvData)
    debugInfo.value = `CSV loaded: ${csvData.length} rows`
    
    if (!csvData?.length) throw new Error('CSV data was empty')
    
    // Extract all unique themes from the CSV data
    const allThemes = csvData.map(row => row.Theme || row.theme)
    const themes = [...new Set(allThemes.flatMap(themeString => themeString.split(';').map(t => t.trim())))]
    
    console.log('All theme strings from CSV:', allThemes)
    console.log('Unique themes after splitting:', themes)
    debugInfo.value = `Found ${themes.length} unique themes and ${csvData.length} events`
    
    const sankeyData = transformCSVToSankey(csvData, themes)
    console.log('Transformed Sankey data:', sankeyData)
    debugInfo.value = `CSV processed: ${sankeyData.nodes.length} nodes (${themes.length} themes + ${csvData.length} events) with ${sankeyData.links.length} total connections`
    
    loading.value = false
    currentData.value = sankeyData
    createSankeyDiagram(sankeyData)
    
  } catch (err) {
    console.error('Error loading CSV data:', err)
    error.value = err.message
    debugInfo.value = `CSV failed: ${err.message}`
    loading.value = false
  }
}

// ===== DATA TRANSFORMATION =====
function transformCSVToSankey(csvData, themes) {
  const sortedData = [...csvData].sort((a, b) => {
    const yearDiff = parseInt(a.Year || a.year) - parseInt(b.Year || b.year)
    return yearDiff !== 0 ? yearDiff : (a.Theme || a.theme).localeCompare(b.Theme || b.theme)
  })
  
  const nodes = [
    // Theme nodes (left side)
    ...themes.map((theme, index) => ({
      id: index,
      name: theme,
      type: 'theme',
      x_position: 0
    })),
    // Event nodes (right side)
    ...sortedData.map((row, index) => ({
      id: themes.length + index,
      name: `${row.Year || row.year} - ${row.Title || row.title}`,
      type: 'event',
      x_position: 1,
      year: parseInt(row.Year || row.year),
      title: row.Title || row.title,
      description: row.Description || row.description
    }))
  ]
  
  // Create links for each theme-event combination
  const links = []
  sortedData.forEach((row, index) => {
    const eventIndex = themes.length + index
    const eventThemes = (row.Theme || row.theme).split(';').map(t => t.trim())
    
    console.log(`Event "${row.Title || row.title}" has themes:`, eventThemes)
    
    eventThemes.forEach(theme => {
      const themeIndex = themes.indexOf(theme)
      if (themeIndex !== -1) {
        links.push({
          source: themeIndex,
          target: eventIndex,
          value: 1,
          description: row.Title || row.title,
          fullDescription: row.Description || row.description,
          year: parseInt(row.Year || row.year),
          theme: theme
        })
      } else {
        console.warn(`Theme "${theme}" not found in themes list:`, themes)
      }
    })
  })
  
  console.log(`Created ${links.length} links for ${sortedData.length} events`)
  
  return { nodes, links }
}

// ===== RESIZE HANDLING =====
function setupResizeObserver() {
  if (!chartContainer.value) return
  
  resizeObserver = new ResizeObserver(() => {
    if (chartContainer.value && !loading.value && currentData.value) {
      createSankeyDiagram(currentData.value)
    }
  })
  resizeObserver.observe(chartContainer.value)
}

// ===== CHART CREATION =====
function createSankeyDiagram(data) {
  if (!data || !chartContainer.value) return
  
  const containerRect = chartContainer.value.getBoundingClientRect()
  const width = Math.max(containerRect.width - CHART_CONFIG.margins.left - CHART_CONFIG.margins.right, CHART_CONFIG.minWidth)
  const height = Math.max(containerRect.height - CHART_CONFIG.margins.top - CHART_CONFIG.margins.bottom, CHART_CONFIG.minHeight)

  d3.select(chartContainer.value).selectAll("*").remove()

  const svg = createSVGContainer(chartContainer.value, width, height, CHART_CONFIG.margins)
  const result = createSankeyLayout(data, width, height)
  const tooltip = createTooltip()

  const { links, nodes, labels } = createChartElements(svg, result, width, tooltip)
  
  createTitle(svg, width)
  addHoverInteractions(links, nodes, labels, width)
}

// ===== SVG CONTAINER CREATION =====
function createSVGContainer(container, width, height, margin) {
  return d3.select(container)
    .append("svg")
    .attr("width", "100%")
    .attr("height", "100%")
    .attr("viewBox", `0 0 ${width + margin.left + margin.right} ${height + margin.top + margin.bottom}`)
    .attr("preserveAspectRatio", "xMidYMid meet")
    .append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`)
}

// ===== SANKEY LAYOUT =====
function createSankeyLayout(data, width, height) {
  return sankey()
    .nodeWidth(CHART_CONFIG.nodeWidth)
    .nodePadding(CHART_CONFIG.nodePadding)
    .nodeSort((a, b) => {
      if (a.x_position !== b.x_position) return a.x_position - b.x_position
      if (a.x_position === 1) return a.year - b.year
      return 0
    })
    .extent([[0, 0], [width, height]])(data)
}

// ===== TOOLTIP CREATION =====
function createTooltip() {
  return d3.select("body").append("div")
    .attr("class", "sankey-tooltip")
    .style("position", "fixed")
    .style("visibility", "hidden")
    .style("background", COLORS.tooltip.bg)
    .style("color", COLORS.text.white)
    .style("padding", "12px")
    .style("border-radius", "6px")
    .style("font-size", "12px")
    .style("font-family", "Arial, sans-serif")
    .style("pointer-events", "none")
    .style("z-index", "1000")
    .style("box-shadow", COLORS.tooltip.shadow)
    .style("max-width", "300px")
    .style("word-wrap", "break-word")
    .style("white-space", "normal")
}

// ===== CHART ELEMENTS CREATION =====
function createChartElements(svg, result, width, tooltip) {
  const links = createLinks(svg, result, tooltip)
  const nodes = createNodes(svg, result, width)
  const labels = createLabels(svg, result, width)
  
  return { links, nodes, labels }
}

function createLinks(svg, result, tooltip) {
  return svg.append("g")
    .selectAll("path")
    .data(result.links)
    .enter().append("path")
    .attr("d", sankeyLinkHorizontal())
    .attr("stroke", d => COLORS.custom[d.source.index % COLORS.custom.length])
    .attr("stroke-width", d => Math.max(4, d.width + 2))
    .attr("fill", "none")
    .attr("opacity", CHART_CONFIG.defaultOpacity)
    .attr("class", "link")
    .style("cursor", "pointer")
    .style("transition", "opacity 0.3s ease")
    .call(addTooltipEvents, tooltip)
}

function createNodes(svg, result, width) {
  return svg.append("g")
    .selectAll("rect")
    .data(result.nodes)
    .enter().append("rect")
    .attr("x", d => d.x0)
    .attr("y", d => d.x0 < width / 2 ? d.y0 : d.y0 - 5)
    .attr("height", d => d.x0 < width / 2 ? d.y1 - d.y0 : (d.y1 - d.y0) + 10)
    .attr("width", d => d.x1 - d.x0)
    .attr("fill", d => d.x0 < width / 2 ? COLORS.custom[d.index % COLORS.custom.length] : COLORS.event)
    .attr("stroke", COLORS.border)
    .attr("stroke-width", 1)
    .attr("rx", d => d.x0 >= width / 2 ? CHART_CONFIG.cornerRadius : 0)
    .attr("ry", d => d.x0 >= width / 2 ? CHART_CONFIG.cornerRadius : 0)
    .attr("class", "node")
    .style("cursor", "pointer")
    .style("transition", "opacity 0.3s ease")
}

function createLabels(svg, result, width) {
  const fontSize = Math.max(CHART_CONFIG.fontSize.min, Math.min(CHART_CONFIG.fontSize.max, width / 80))
  
  const themeLabels = createThemeLabels(svg, result, width, fontSize)
  const yearLabels = createYearLabels(svg, result, width, fontSize)
  const titleLabels = createTitleLabels(svg, result, width, fontSize)
  
  return { themeLabels, yearLabels, titleLabels }
}

function createThemeLabels(svg, result, width, fontSize) {
  return svg.append("g")
    .selectAll("text")
    .data(result.nodes.filter(d => d.x0 < width / 2))
    .enter().append("text")
    .attr("x", d => d.x1 + 6)
    .attr("y", d => (d.y1 + d.y0) / 2)
    .attr("dy", "0.35em")
    .attr("text-anchor", "start")
    .text(d => d.name)
    .attr("font-size", `${fontSize}px`)
    .attr("fill", COLORS.text.primary)
    .attr("font-weight", "bold")
    .attr("class", "label")
    .style("pointer-events", "none")
}

function createYearLabels(svg, result, width, fontSize) {
  return svg.append("g")
    .selectAll("text")
    .data(result.nodes.filter(d => d.x0 >= width / 2))
    .enter().append("text")
    .attr("x", d => (d.x0 + d.x1) / 2)
    .attr("y", d => (d.y1 + d.y0 - 5) / 2 + 5)
    .attr("dy", "0.35em")
    .attr("text-anchor", "middle")
    .text(d => d.year)
    .attr("font-size", `${fontSize - 2}px`)
    .attr("fill", COLORS.text.white)
    .attr("font-weight", "bold")
    .attr("class", "label")
    .style("pointer-events", "none")
}

function createTitleLabels(svg, result, width, fontSize) {
  return svg.append("g")
    .selectAll("text")
    .data(result.nodes.filter(d => d.x0 >= width / 2))
    .enter().append("text")
    .attr("x", d => d.x1 + 6)
    .attr("y", d => (d.y1 + d.y0 - 5) / 2 + 5)
    .attr("dy", "0.35em")
    .attr("text-anchor", "start")
    .text(d => d.title)
    .attr("font-size", `${fontSize - 3}px`)
    .attr("fill", COLORS.text.primary)
    .attr("font-weight", "bold")
    .attr("class", "label")
    .style("pointer-events", "none")
}

// ===== TOOLTIP EVENTS =====
function addTooltipEvents(selection, tooltip) {
  selection
    .on("mouseenter", (event, d) => showTooltip(event, d, tooltip))
    .on("mousemove", (event) => updateTooltipPosition(event, tooltip))
    .on("mouseleave", function() {
      d3.select(this).style("opacity", CHART_CONFIG.defaultOpacity)
      tooltip.style("visibility", "hidden")
    })
}

function showTooltip(event, d, tooltip) {
  d3.select(event.currentTarget)
    .style("opacity", CHART_CONFIG.linkOpacityHover)
    .style("stroke", COLORS.custom[d.source.index % COLORS.custom.length])
  
  tooltip.style("visibility", "visible")
    .html(`
      <div style="font-weight: bold; margin-bottom: 8px; font-size: 14px; word-wrap: break-word;">${d.description}</div>
      <div style="font-size: 11px; opacity: 0.9; margin-bottom: 4px;"><strong>Theme:</strong> ${d.theme}</div>
      <div style="font-size: 11px; opacity: 0.9; margin-bottom: 4px;"><strong>Year:</strong> ${d.year}</div>
      <div style="font-size: 10px; opacity: 0.8; line-height: 1.3; word-wrap: break-word;">${d.fullDescription}</div>
    `)
    .style("left", (event.clientX + CHART_CONFIG.tooltipOffset) + "px")
    .style("top", (event.clientY - CHART_CONFIG.tooltipOffset) + "px")
}

function updateTooltipPosition(event, tooltip) {
  tooltip
    .style("left", (event.clientX + CHART_CONFIG.tooltipOffset) + "px")
    .style("top", (event.clientY - CHART_CONFIG.tooltipOffset) + "px")
}

// ===== TITLE CREATION =====
function createTitle(svg, width) {
  const titleFontSize = Math.max(CHART_CONFIG.fontSize.title.min, Math.min(CHART_CONFIG.fontSize.title.max, width / 50))
  svg.append("text")
    .attr("x", width / 2)
    .attr("y", -5)
    .attr("text-anchor", "middle")
    .style("font-size", `${titleFontSize}px`)
    .style("font-weight", "bold")
    .text(prismicData.value?.title || "Sankey Diagram")
}

// ===== HOVER INTERACTIONS =====
function addHoverInteractions(links, nodes, labels, width) {
  const isThemeNode = d => d.x0 < width / 2
  
  nodes.filter(isThemeNode)
    .on("mouseenter", (event, d) => highlightThemeFlows(d.index, links, nodes, labels, width))
    .on("mouseleave", () => resetHighlighting(links, nodes, labels))

  nodes.filter(d => !isThemeNode(d))
    .on("mouseenter", (event, d) => highlightEventFlows(d.index, links, nodes, labels, width))
    .on("mouseleave", () => resetHighlighting(links, nodes, labels))
}

function highlightThemeFlows(themeIndex, links, nodes, labels, width) {
  links.style("opacity", CHART_CONFIG.backgroundOpacity)
  
  // Highlight all links from this theme
  links.filter(link => link.source.index === themeIndex)
    .style("opacity", CHART_CONFIG.nodeOpacityHighlight)
    .style("stroke", COLORS.custom[themeIndex % COLORS.custom.length])
  
  nodes.filter(node => node.x0 < width / 2 && node.index !== themeIndex)
    .style("opacity", CHART_CONFIG.backgroundOpacity)
  
  nodes.filter(node => node.x0 >= width / 2)
    .style("opacity", CHART_CONFIG.backgroundOpacity)
  
  labels.themeLabels.filter(label => label.index !== themeIndex)
    .style("opacity", CHART_CONFIG.backgroundOpacity)
}

function highlightEventFlows(eventIndex, links, nodes, labels, width) {
  links.style("opacity", CHART_CONFIG.backgroundOpacity)
  
  // Highlight all links connected to this event
  links.filter(link => link.target.index === eventIndex)
    .style("opacity", CHART_CONFIG.nodeOpacityHighlight)
  
  nodes.filter(node => node.x0 >= width / 2 && node.index !== eventIndex)
    .style("opacity", CHART_CONFIG.backgroundOpacity)
  
  nodes.filter(node => node.x0 < width / 2)
    .style("opacity", CHART_CONFIG.backgroundOpacity)
  
  labels.yearLabels.filter(label => label.index !== eventIndex)
    .style("opacity", CHART_CONFIG.backgroundOpacity)
  labels.titleLabels.filter(label => label.index !== eventIndex)
    .style("opacity", CHART_CONFIG.backgroundOpacity)
}

function resetHighlighting(links, nodes, labels) {
  // Reset links to their default opacity
  links.style("opacity", CHART_CONFIG.defaultOpacity)
  
  // Reset nodes to their default opacity
  nodes.style("opacity", CHART_CONFIG.defaultOpacity)
  
  // Reset all labels to full opacity
  labels.themeLabels.style("opacity", 1)
  labels.yearLabels.style("opacity", 1)
  labels.titleLabels.style("opacity", 1)
}
</script>

<style scoped>
.diagram {
  padding: 20px;
  font-family: Arial, sans-serif;
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.prismic-header {
  margin-bottom: 20px;
  text-align: center;
}

.prismic-header h1 {
  color: #333;
  font-size: 2.5rem;
  margin: 0;
  font-weight: 300;
}

.chart-container {
  flex: 1;
  min-height: 600px;
  border: 1px solid #ccc;
  border-radius: 8px;
  background: #fafafa;
  overflow: auto;
}

.loading {
  text-align: center;
  padding: 20px;
  color: #666;
  font-style: italic;
}

.error {
  text-align: center;
  padding: 20px;
  color: #d32f2f;
  background: #ffebee;
  border: 1px solid #ffcdd2;
  border-radius: 4px;
  margin: 10px;
}

.debug-info {
  text-align: center;
  padding: 10px;
  color: #555;
  font-size: 0.9em;
  margin-top: 10px;
}

@media (max-width: 768px) {
  .diagram { padding: 10px; }
  .chart-container { min-height: 400px; }
  .prismic-header h1 { font-size: 2rem; }
}

@media (max-width: 480px) {
  .diagram { padding: 5px; }
  .chart-container { min-height: 350px; }
  .prismic-header h1 { font-size: 1.5rem; }
}
</style>
