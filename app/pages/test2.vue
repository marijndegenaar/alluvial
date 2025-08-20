<template>
  <div class="diagram">
    <h1>Eduvia Educational History: Themes to Years (1900-1955)</h1>
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

// ===== REACTIVE STATE =====
const chartContainer = ref(null)        // Reference to the DOM element where the chart will be drawn
const loading = ref(true)               // Loading state for the UI
const error = ref(null)                 // Error state for displaying any issues
const debugInfo = ref('Initializing...') // Debug information for troubleshooting
const currentData = ref(null)           // Stores the processed Sankey data

// ===== RESIZE OBSERVER =====
let resizeObserver = null               // Watches for container size changes to redraw chart

// ===== LIFECYCLE HOOKS =====
onMounted(() => {
    debugInfo.value = 'Component mounted, starting data load...'
    loadData()                          // Load CSV data when component mounts
    setupResizeObserver()               // Set up responsive behavior
})

onUnmounted(() => {
    if (resizeObserver) resizeObserver.disconnect()  // Clean up observer
})

// ===== RESIZE HANDLING =====
function setupResizeObserver() {
    if (chartContainer.value) {
        // Create observer that redraws chart when container size changes
        resizeObserver = new ResizeObserver(() => {
            if (chartContainer.value && !loading.value && currentData.value) {
                createSankeyDiagram(currentData.value)
            }
        })
        resizeObserver.observe(chartContainer.value)
    }
}

// ===== DATA LOADING =====
async function loadData() {
    try {
        loading.value = true
        error.value = null
        debugInfo.value = 'Loading sample site data...'
        
        // Fetch CSV data from the public folder
        const csvData = await d3.csv('/data/sample_site_data_overlapping.csv')
        console.log('Raw CSV data:', csvData)
        debugInfo.value = `CSV loaded: ${csvData.length} rows`
        
        if (!csvData?.length) throw new Error('CSV data was empty')
        
        // Extract unique values from CSV columns
        const themes = [...new Set(csvData.map(row => row.Theme))]  // Get unique themes
        const years = [...new Set(csvData.map(row => row.Year))]    // Get unique years
        
        console.log('Unique themes:', themes)
        console.log('Unique years:', years)
        debugInfo.value = `Found ${themes.length} themes and ${years.length} years`
        
        // Transform CSV data into Sankey-compatible format
        const sankeyData = transformCSVToSankey(csvData, themes, years)
        console.log('Transformed Sankey data:', sankeyData)
        debugInfo.value = 'CSV data processed successfully'
        
        // Update state and create the chart
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
function transformCSVToSankey(csvData, themes, years) {
    // Create nodes array: themes on left (x_position: 0), years on right (x_position: 1)
    const nodes = [
        // Left side: theme nodes with IDs 0, 1, 2, 3, 4...
        ...themes.map((theme, index) => ({
            id: index,                   // Numeric ID for D3-Sankey
            name: theme,                 // Display name
            type: 'theme',               // Node type for styling
            x_position: 0                // Left side of diagram
        })),
        // Right side: year nodes with IDs 5, 6, 7, 8... (continuing from themes)
        ...years.map((year, index) => ({
            id: themes.length + index,   // ID continues from where themes left off
            name: year.toString(),       // Display name
            type: 'year',                // Node type for styling
            x_position: 1                // Right side of diagram
        }))
    ]
    
    // Create links array: each CSV row becomes a flow from theme to year
    const links = csvData.map(row => {
        const themeIndex = themes.indexOf(row.Theme)      // Find theme's position
        const yearIndex = years.indexOf(row.Year)         // Find year's position
        return {
            source: themeIndex,                           // Source node ID (theme)
            target: themes.length + yearIndex,            // Target node ID (year)
            value: 1,                                     // Flow weight (all equal for now)
            description: row.Title,                       // Event title for tooltip
            fullDescription: row.Description,             // Full description for tooltip
            year: row.Year,                               // Year for tooltip
            theme: row.Theme                              // Theme for tooltip
        }
    })
    
    return { nodes, links }
}

// ===== CHART CREATION =====
function createSankeyDiagram(data) {
    if (!data || !chartContainer.value) return
    
    // Calculate chart dimensions based on container size
    const containerRect = chartContainer.value.getBoundingClientRect()
    const margin = { top: 20, right: 20, bottom: 20, left: 20 }
    const width = Math.max(containerRect.width - margin.left - margin.right, 300)   // Minimum 300px
    const height = Math.max(containerRect.height - margin.top - margin.bottom, 400) // Minimum 400px

    // Clear any existing chart
    d3.select(chartContainer.value).selectAll("*").remove()

    // Create SVG container and apply Sankey layout
    const svg = createSVGContainer(chartContainer.value, width, height, margin)
    const result = createSankeyLayout(data, width, height)
    const tooltip = createTooltip()

    // Create all chart elements (links, nodes, labels)
    const { links, nodes, labels } = createChartElements(svg, result, width, tooltip)
    
    // Add title and interactive behaviors
    createTitle(svg, width)
    addHoverInteractions(links, nodes, labels, result, width)
}

// ===== SVG CONTAINER CREATION =====
function createSVGContainer(container, width, height, margin) {
    return d3.select(container)
        .append("svg")
        .attr("width", "100%")                          // Responsive width
        .attr("height", "100%")                         // Responsive height
        .attr("viewBox", `0 0 ${width + margin.left + margin.right} ${height + margin.top + margin.bottom}`)
        .attr("preserveAspectRatio", "xMidYMid meet")   // Maintain aspect ratio
        .append("g")                                    // Group for chart content
        .attr("transform", `translate(${margin.left},${margin.top})`)  // Apply margins
}

// ===== SANKEY LAYOUT =====
function createSankeyLayout(data, width, height) {
    // Configure and apply D3-Sankey algorithm
    return sankey()
        .nodeWidth(25)                                  // Width of each node
        .nodePadding(20)                                // Space between nodes
        .extent([[0, 0], [width, height]])(data)       // Chart boundaries
}

// ===== TOOLTIP CREATION =====
function createTooltip() {
    // Create floating tooltip div that shows on hover
    return d3.select("body").append("div")
        .attr("class", "sankey-tooltip")
        .style("position", "absolute")
        .style("visibility", "hidden")
        .style("background", "rgba(0,0,0,0.9)")
        .style("color", "white")
        .style("padding", "12px")
        .style("border-radius", "6px")
        .style("font-size", "12px")
        .style("font-family", "Arial, sans-serif")
        .style("pointer-events", "none")                // Don't interfere with mouse events
        .style("z-index", "1000")                       // Ensure tooltip appears above chart
        .style("box-shadow", "0 4px 8px rgba(0,0,0,0.3)")
        .style("max-width", "300px")
}

// ===== CHART ELEMENTS CREATION =====
function createChartElements(svg, result, width, tooltip) {
    // Create flow paths (the curved lines between nodes)
    const links = svg.append("g")
        .selectAll("path")
        .data(result.links)                             // Bind data to paths
        .enter().append("path")                         // Create new paths for each link
        .attr("d", sankeyLinkHorizontal())              // Use D3's horizontal link generator
        .attr("stroke", d => d3.schemeCategory10[d.source.index % 10])  // Color by source theme
        .attr("stroke-width", d => Math.max(2, d.width)) // Width based on flow value
        .attr("fill", "none")                           // No fill, just stroke
        .attr("opacity", 0.7)                           // Semi-transparent
        .attr("class", "link")                          // CSS class for styling
        .style("cursor", "pointer")                     // Show pointer on hover
        .style("transition", "opacity 0.3s ease")       // Smooth opacity changes
        .call(addTooltipEvents, tooltip)                // Add tooltip behavior

    // Create node rectangles (the boxes representing themes and years)
    const nodes = svg.append("g")
        .selectAll("rect")
        .data(result.nodes)                             // Bind data to rectangles
        .enter().append("rect")                         // Create new rectangles for each node
        .attr("x", d => d.x0)                           // X position from Sankey layout
        .attr("y", d => d.y0)                           // Y position from Sankey layout
        .attr("height", d => d.y1 - d.y0)               // Height from Sankey layout
        .attr("width", d => d.x1 - d.x0)                // Width from Sankey layout
        .attr("fill", d => d.x0 < width / 2 ? d3.schemeCategory10[d.index % 10] : "#2E8B57")  // Color: themes=colorful, years=green
        .attr("stroke", "#000")                         // Black border
        .attr("stroke-width", 1)                        // Thin border
        .attr("class", "node")                          // CSS class for styling
        .style("cursor", "pointer")                     // Show pointer on hover
        .style("transition", "opacity 0.3s ease")       // Smooth opacity changes

    // Create text labels for nodes
    const fontSize = Math.max(10, Math.min(14, width / 80))  // Responsive font size
    const labels = svg.append("g")
        .selectAll("text")
        .data(result.nodes)                             // Bind data to text elements
        .enter().append("text")                         // Create new text for each node
        .attr("x", d => d.x0 < width / 2 ? d.x1 + 6 : d.x0 - 6)  // Position: themes=right of node, years=left of node
        .attr("y", d => (d.y1 + d.y0) / 2)             // Center vertically on node
        .attr("dy", "0.35em")                           // Fine-tune vertical alignment
        .attr("text-anchor", d => d.x0 < width / 2 ? "start" : "end")  // Text alignment: themes=left-aligned, years=right-aligned
        .text(d => d.name)                              // Display node name
        .attr("font-size", `${fontSize}px`)             // Set font size
        .attr("fill", "#333")                           // Dark gray text
        .attr("font-weight", "bold")                    // Bold text
        .attr("class", "label")                         // CSS class for styling
        .style("pointer-events", "none")                // Don't interfere with node hover

    return { links, nodes, labels }
}

// ===== TOOLTIP EVENTS =====
function addTooltipEvents(selection, tooltip) {
    selection
        .on("mouseenter", function(event, d) {
            // Highlight the hovered link
            d3.select(this)
                .style("opacity", 1)
                .style("stroke-width", d => Math.max(4, d.width + 2))  // Make it thicker and more opaque
                .style("stroke", d3.schemeCategory10[d.source.index % 10])  // Ensure full color
            
            // Show tooltip with event details
            tooltip.style("visibility", "visible")
                .html(`
                    <div style="font-weight: bold; margin-bottom: 8px; font-size: 14px;">${d.description}</div>
                    <div style="font-size: 11px; opacity: 0.9; margin-bottom: 4px;"><strong>Theme:</strong> ${d.theme}</div>
                    <div style="font-size: 11px; opacity: 0.9; margin-bottom: 4px;"><strong>Year:</strong> ${d.year}</div>
                    <div style="font-size: 10px; opacity: 0.8; line-height: 1.3;">${d.fullDescription}</div>
                `)
                .style("left", (event.pageX + 15) + "px")  // Position to right of cursor
                .style("top", (event.pageY - 15) + "px")   // Position above cursor
        })
        .on("mousemove", function(event, d) {
            // Update tooltip position as mouse moves
            tooltip.style("left", (event.pageX + 15) + "px")
                .style("top", (event.pageY - 15) + "px")
        })
        .on("mouseleave", function() {
            // Reset link appearance when mouse leaves
            d3.select(this)
                .style("opacity", 0.7)
                .style("stroke-width", d => Math.max(2, d.width))
            
            // Hide tooltip
            tooltip.style("visibility", "hidden")
        })
}

// ===== TITLE CREATION =====
function createTitle(svg, width) {
    const titleFontSize = Math.max(14, Math.min(18, width / 50))  // Responsive font size
    svg.append("text")
        .attr("x", width / 2)                           // Center horizontally
        .attr("y", -5)                                  // Position above chart
        .attr("text-anchor", "middle")                  // Center-align text
        .style("font-size", `${titleFontSize}px`)       // Set font size
        .style("font-weight", "bold")                   // Bold text
        .text("Eduvia Educational History: Themes to Years (1900-1955)")
}

// ===== HOVER INTERACTIONS =====
function addHoverInteractions(links, nodes, labels, result, width) {
    const isLeftSide = d => d.x0 < width / 2  // Helper function to identify theme nodes
    
    // Add hover effects to theme nodes (left side)
    nodes.filter(isLeftSide)
        .on("mouseenter", function(event, d) {
            highlightThemeFlows(d.index, links, nodes, labels, result, width)
        })
        .on("mouseleave", () => resetHighlighting(links, nodes, labels))

    // Add hover effects to year nodes (right side)
    nodes.filter(d => !isLeftSide(d))
        .on("mouseenter", function(event, d) {
            highlightYearFlows(d.index, links, nodes, labels, result, width)
        })
        .on("mouseleave", () => resetHighlighting(links, nodes, labels))
}

// ===== THEME HOVER HIGHLIGHTING =====
function highlightThemeFlows(themeIndex, links, nodes, labels, result, width) {
    // Fade out all links
    links.style("opacity", 0.1)
    
    // Highlight links from the hovered theme
    links.filter(link => link.source.index === themeIndex)
        .style("opacity", 0.9)
        .style("stroke-width", d => Math.max(3, d.width + 1))  // Make them thicker
    
    // Fade out other theme nodes
    nodes.filter(node => node.x0 < width / 2 && node.index !== themeIndex)
        .style("opacity", 0.3)
    
    // Keep year nodes visible but dimmed
    nodes.filter(node => node.x0 >= width / 2)
        .style("opacity", 0.7)
    
    // Fade out labels for other themes
    labels.filter(label => label.x < width / 2 && label.index !== themeIndex)
        .style("opacity", 0.3)
}

// ===== YEAR HOVER HIGHLIGHTING =====
function highlightYearFlows(yearIndex, links, nodes, labels, result, width) {
    // Fade out all links
    links.style("opacity", 0.1)
    
    // Highlight links to the hovered year
    links.filter(link => link.target.index === yearIndex)
        .style("opacity", 0.9)
        .style("stroke-width", d => Math.max(3, d.width + 1))  // Make them thicker
    
    // Fade out other year nodes
    nodes.filter(node => node.x0 >= width / 2 && node.index !== yearIndex)
        .style("opacity", 0.3)
    
    // Keep theme nodes visible but dimmed
    nodes.filter(node => node.x0 < width / 2)
        .style("opacity", 0.7)
    
    // Fade out labels for other years
    labels.filter(label => label.x >= width / 2 && label.index !== yearIndex)
        .style("opacity", 0.3)
}

// ===== RESET HIGHLIGHTING =====
function resetHighlighting(links, nodes, labels) {
    // Restore all elements to their normal appearance
    links.style("opacity", 0.7)
        .style("stroke-width", d => Math.max(2, d.width))
    nodes.style("opacity", 1)
    labels.style("opacity", 1)
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

h1 {
    text-align: center;
    color: #333;
    margin-bottom: 20px;
    flex-shrink: 0;
}

.chart-container {
    flex: 1;
    min-height: 500px;
    border: 1px solid #ccc;
    border-radius: 8px;
    background: #fafafa;
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
    h1 { font-size: 1.5rem; margin-bottom: 15px; }
    .chart-container { min-height: 400px; }
}

@media (max-width: 480px) {
    .diagram { padding: 5px; }
    h1 { font-size: 1.2rem; margin-bottom: 10px; }
    .chart-container { min-height: 350px; }
}
</style>
