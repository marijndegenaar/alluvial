<template>
  <div class="diagram">
    <h1>Historical Themes Flow Through Decades (1900-1950)</h1>
    <div ref="chartContainer" class="chart-container"></div>
    <div v-if="loading" class="loading">Loading data...</div>
    <div v-if="error" class="error">Error loading data: {{ error }}</div>
  </div>
</template>

<script setup>
import * as d3 from 'd3'
import { sankey, sankeyLinkHorizontal } from 'd3-sankey'
import { onMounted, ref, onUnmounted } from 'vue'

// Reactive references
const chartContainer = ref(null)
const loading = ref(true)
const error = ref(null)
const currentData = ref(null)

// Resize observer for responsive behavior
let resizeObserver = null

onMounted(() => {
    loadData()
    setupResizeObserver()
})

onUnmounted(() => {
    if (resizeObserver) {
        resizeObserver.disconnect()
    }
})

function setupResizeObserver() {
    if (chartContainer.value) {
        resizeObserver = new ResizeObserver(() => {
            if (chartContainer.value && !loading.value && currentData.value) {
                createSankeyDiagram(currentData.value)
            }
        })
        resizeObserver.observe(chartContainer.value)
    }
}

async function loadData() {
    try {
        loading.value = true
        error.value = null
        
        const csvData = await d3.csv('/data/historical-data.csv')
        
        if (csvData?.length > 0) {
            const { nodes, links } = parseCSVData(csvData)
            
            if (nodes.length > 0 && links.length > 0) {
                const sankeyData = transformData(nodes, links)
                loading.value = false
                currentData.value = sankeyData
                createSankeyDiagram(sankeyData)
                return
            }
        }
        
        throw new Error('Invalid or empty CSV data')
        
    } catch (err) {
        console.error('Error loading CSV data:', err)
        error.value = err.message
        useFallbackData()
    }
}

function parseCSVData(csvData) {
    const nodes = csvData.filter(row => row.type === 'node')
    const links = csvData.filter(row => row.type === 'link')
    
    return { nodes, links }
}

function transformData(nodes, links) {
    return {
        nodes: nodes.map(node => ({
            id: parseInt(node.id),
            name: node.name,
            type: node.x_position === '0' ? 'theme' : 'decade',
            x_position: parseInt(node.x_position)
        })),
        links: links.map(link => ({
            source: parseInt(link.source),
            target: parseInt(link.target),
            value: parseInt(link.value),
            description: link.description || ''
        }))
    }
}

function useFallbackData() {
    const fallbackData = {
        nodes: [
            { id: 0, name: "Industrial Revolution", type: "theme", x_position: 0 },
            { id: 1, name: "World Wars", type: "theme", x_position: 0 },
            { id: 2, name: "Colonialism", type: "theme", x_position: 0 },
            { id: 3, name: "Economic Crisis", type: "theme", x_position: 0 },
            { id: 4, name: "1900-1910", type: "decade", x_position: 1 },
            { id: 5, name: "1910-1920", type: "decade", x_position: 1 },
            { id: 6, name: "1920-1930", type: "decade", x_position: 1 },
            { id: 7, name: "1930-1940", type: "decade", x_position: 1 },
            { id: 8, name: "1940-1950", type: "decade", x_position: 1 }
        ],
        links: [
            { source: 0, target: 4, value: 25, description: "Industrial Revolution to 1900s" },
            { source: 0, target: 5, value: 20, description: "Industrial Revolution to 1910s" },
            { source: 0, target: 6, value: 15, description: "Industrial Revolution to 1920s" },
            { source: 1, target: 5, value: 30, description: "World Wars to 1910s" },
            { source: 1, target: 7, value: 35, description: "World Wars to 1930s" },
            { source: 1, target: 8, value: 40, description: "World Wars to 1940s" },
            { source: 2, target: 4, value: 20, description: "Colonialism to 1900s" },
            { source: 2, target: 5, value: 18, description: "Colonialism to 1910s" },
            { source: 2, target: 6, value: 15, description: "Colonialism to 1920s" },
            { source: 2, target: 7, value: 12, description: "Colonialism to 1930s" },
            { source: 2, target: 8, value: 8, description: "Colonialism to 1940s" },
            { source: 3, target: 6, value: 25, description: "Economic Crisis to 1920s" },
            { source: 3, target: 7, value: 45, description: "Economic Crisis to 1930s" },
            { source: 3, target: 8, value: 20, description: "Economic Crisis to 1940s" }
        ]
    }
    
    loading.value = false
    currentData.value = fallbackData
    createSankeyDiagram(fallbackData)
}

function createSankeyDiagram(data) {
    if (!data || !chartContainer.value) return
    
    // Get container dimensions
    const containerRect = chartContainer.value.getBoundingClientRect()
    const margin = { top: 20, right: 20, bottom: 20, left: 20 }
    const width = Math.max(containerRect.width - margin.left - margin.right, 300)
    const height = Math.max(containerRect.height - margin.top - margin.bottom, 400)

    // Clear any existing content
    d3.select(chartContainer.value).selectAll("*").remove()

    // Create SVG container
    const svg = createSVGContainer(chartContainer.value, width, height, margin)
    
    // Create Sankey layout
    const result = createSankeyLayout(data, width, height)
    
    // Draw chart elements
    drawLinks(svg, result.links, width)
    drawNodes(svg, result.nodes, width)
    drawLabels(svg, result.nodes, width)
    drawFlowLabels(svg, result.links)
    drawTitle(svg, width)
    
    // Add interactivity
    addHoverInteractions(svg, result, width)
}

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

function createSankeyLayout(data, width, height) {
    // Calculate dynamic dimensions based on data
    const nodeCount = data.nodes.length
    const linkCount = data.links.length
    
    // Adjust spacing based on data volume
    const dynamicNodePadding = Math.max(10, 50 - nodeCount * 2)
    const dynamicHeight = Math.max(height, nodeCount * 60)
    
    const sankeyLayout = sankey()
        .nodeWidth(25)
        .nodePadding(dynamicNodePadding)
        .extent([[0, 0], [width, dynamicHeight]])
    
    return sankeyLayout(data)
}

function drawLinks(svg, links, width) {
    svg.append("g")
        .selectAll("path")
        .data(links)
        .enter().append("path")
        .attr("d", sankeyLinkHorizontal())
        .attr("stroke", d => d3.schemeCategory10[d.source.index % 10])
        .attr("stroke-width", d => Math.max(2, d.width))
        .attr("fill", "none")
        .attr("opacity", 0.7)
        .attr("class", "link")
        .style("cursor", "pointer")
        .style("transition", "opacity 0.3s ease")
}

function drawNodes(svg, nodes, width) {
    svg.append("g")
        .selectAll("rect")
        .data(nodes)
        .enter().append("rect")
        .attr("x", d => d.x0)
        .attr("y", d => d.y0)
        .attr("height", d => d.y1 - d.y0)
        .attr("width", d => d.x1 - d.x0)
        .attr("fill", d => d.x0 < width / 2 ? d3.schemeCategory10[d.index % 10] : "#2E8B57")
        .attr("stroke", "#000")
        .attr("stroke-width", 1)
        .attr("class", "node")
        .style("cursor", "pointer")
        .style("transition", "opacity 0.3s ease")
}

function drawLabels(svg, nodes, width) {
    const fontSize = Math.max(10, Math.min(14, width / 80))
    
    svg.append("g")
        .selectAll("text")
        .data(nodes)
        .enter().append("text")
        .attr("x", d => d.x0 < width / 2 ? d.x1 + 6 : d.x0 - 6)
        .attr("y", d => (d.y1 + d.y0) / 2)
        .attr("dy", "0.35em")
        .attr("text-anchor", d => d.x0 < width / 2 ? "start" : "end")
        .text(d => d.name)
        .attr("font-size", `${fontSize}px`)
        .attr("fill", "#333")
        .attr("font-weight", "bold")
        .attr("class", "label")
        .style("pointer-events", "none")
}

function drawFlowLabels(svg, links) {
    svg.append("g")
        .selectAll("text")
        .data(links)
        .enter().append("text")
        .attr("x", d => (d.source.x1 + d.target.x0) / 2)
        .attr("y", d => (d.source.y1 + d.target.y0) / 2)
        .attr("dy", "0.35em")
        .attr("text-anchor", "middle")
        .text(d => d.value)
        .attr("font-size", "9px")
        .attr("fill", "#666")
        .attr("font-weight", "bold")
        .attr("class", "flow-label")
        .style("pointer-events", "none")
}

function drawTitle(svg, width) {
    svg.append("text")
        .attr("x", width / 2)
        .attr("y", -5)
        .attr("text-anchor", "middle")
        .style("font-size", "16px")
        .style("font-weight", "bold")
        .text("Historical Themes Flow Through Decades (1900-1950)")
}

function addHoverInteractions(svg, result, width) {
    const links = svg.selectAll(".link")
    const nodes = svg.selectAll(".node")
    const labels = svg.selectAll(".label")
    const flowLabels = svg.selectAll(".flow-label")
    
    // Theme nodes hover (left side)
    nodes.filter(d => d.x0 < width / 2)
        .on("mouseenter", function(event, d) {
            highlightThemeFlows(d.index, links, nodes, labels, flowLabels, result, width)
        })
        .on("mouseleave", function() {
            resetHighlighting(links, nodes, labels, flowLabels)
        })

    // Decade nodes hover (right side)
    nodes.filter(d => d.x0 >= width / 2)
        .on("mouseenter", function(event, d) {
            highlightDecadeFlows(d.index, links, nodes, labels, flowLabels, result, width)
        })
        .on("mouseleave", function() {
            resetHighlighting(links, nodes, labels, flowLabels)
        })
}

function highlightThemeFlows(themeIndex, links, nodes, labels, flowLabels, result, width) {
    links.style("opacity", 0.1)
    links.filter(link => link.source.index === themeIndex)
        .style("opacity", 0.9)
        .style("stroke-width", d => Math.max(3, d.width + 1))
    
    nodes.filter(node => node.x0 < width / 2 && node.index !== themeIndex)
        .style("opacity", 0.3)
    nodes.filter(node => node.x0 >= width / 2)
        .style("opacity", 0.7)
    
    labels.filter(label => label.x < width / 2 && label.index !== themeIndex)
        .style("opacity", 0.3)
    flowLabels.filter((label, i) => result.links[i].source.index !== themeIndex)
        .style("opacity", 0.1)
}

function highlightDecadeFlows(decadeIndex, links, nodes, labels, flowLabels, result, width) {
    links.style("opacity", 0.1)
    links.filter(link => link.target.index === decadeIndex)
        .style("opacity", 0.9)
        .style("stroke-width", d => Math.max(3, d.width + 1))
    
    nodes.filter(node => node.x0 >= width / 2 && node.index !== decadeIndex)
        .style("opacity", 0.3)
    nodes.filter(node => node.x0 < width / 2)
        .style("opacity", 0.7)
    
    labels.filter(label => label.x >= width / 2 && label.index !== decadeIndex)
        .style("opacity", 0.3)
    flowLabels.filter((label, i) => result.links[i].target.index !== decadeIndex)
        .style("opacity", 0.1)
}

function resetHighlighting(links, nodes, labels, flowLabels) {
    links.style("opacity", 0.7)
        .style("stroke-width", d => Math.max(2, d.width))
    nodes.style("opacity", 1)
    labels.style("opacity", 1)
    flowLabels.style("opacity", 1)
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

/* Responsive breakpoints */
@media (max-width: 768px) {
    .diagram {
        padding: 10px;
    }
    
    h1 {
        font-size: 1.5rem;
        margin-bottom: 15px;
    }
    
    .chart-container {
        min-height: 400px;
    }
}

@media (max-width: 480px) {
    .diagram {
        padding: 5px;
    }
    
    h1 {
        font-size: 1.2rem;
        margin-bottom: 10px;
    }
    
    .chart-container {
        min-height: 350px;
    }
}
</style>
