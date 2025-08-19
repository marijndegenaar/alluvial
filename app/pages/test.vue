<template>
    <div>
        <h1>Hello World</h1>
        <p ref="d3Target"></p>
        <div ref="alluvialContainer" style="width: 100%; height: 600px; border: 1px solid #ccc; margin: 20px 0;"></div>
    </div>
</template>

<script setup>
import * as d3 from 'd3'
import { onMounted, ref } from 'vue'

const d3Target = ref(null)
const alluvialContainer = ref(null)

onMounted(() => {
    if (d3Target.value) {
        d3.select(d3Target.value).text("This text is manipulated by d3.js")
    }
    
    if (alluvialContainer.value) {
        createAlluvialDiagram()
    }
})

function createAlluvialDiagram() {
    // Sample data for the alluvial diagram
    const data = [
        { source: "Desktop", target: "Chrome", value: 45 },
        { source: "Desktop", target: "Firefox", value: 25 },
        { source: "Desktop", target: "Safari", value: 20 },
        { source: "Mobile", target: "Chrome", value: 35 },
        { source: "Mobile", target: "Safari", value: 30 },
        { source: "Tablet", target: "Chrome", value: 15 },
        { source: "Tablet", target: "Safari", value: 10 }
    ]

    const margin = { top: 20, right: 20, bottom: 20, left: 20 }
    const width = alluvialContainer.value.clientWidth - margin.left - margin.right
    const height = 500 - margin.top - margin.bottom

    // Create SVG
    const svg = d3.select(alluvialContainer.value)
        .append("svg")
        .attr("width", width + margin.left + margin.right)
        .attr("height", height + margin.top + margin.bottom)
        .append("g")
        .attr("transform", `translate(${margin.left},${margin.top})`)

    // Extract unique sources and targets
    const sources = [...new Set(data.map(d => d.source))]
    const targets = [...new Set(data.map(d => d.target))]

    // Create scales
    const sourceScale = d3.scaleBand()
        .domain(sources)
        .range([0, width * 0.4])
        .padding(0.1)

    const targetScale = d3.scaleBand()
        .domain(targets)
        .range([width * 0.6, width])
        .padding(0.1)

    const yScale = d3.scaleLinear()
        .domain([0, d3.max(data, d => d.value)])
        .range([0, height * 0.8])

    // Draw source nodes
    svg.selectAll(".source-node")
        .data(sources)
        .enter()
        .append("rect")
        .attr("class", "source-node")
        .attr("x", 0)
        .attr("y", (d, i) => sourceScale(d) + sourceScale.bandwidth() / 2 - 10)
        .attr("width", 20)
        .attr("height", 20)
        .attr("fill", "#69b3a2")
        .attr("rx", 3)

    // Draw target nodes
    svg.selectAll(".target-node")
        .data(targets)
        .enter()
        .append("rect")
        .attr("class", "target-node")
        .attr("x", width - 20)
        .attr("y", (d, i) => targetScale(d) + targetScale.bandwidth() / 2 - 10)
        .attr("width", 20)
        .attr("height", 20)
        .attr("fill", "#ff7f0e")
        .attr("rx", 3)

    // Draw flows (alluvial paths)
    data.forEach(d => {
        const sourceY = sourceScale(d.source) + sourceScale.bandwidth() / 2
        const targetY = targetScale(d.target) + targetScale.bandwidth() / 2
        const flowHeight = yScale(d.value)

        // Create curved path
        const path = d3.path()
        path.moveTo(20, sourceY)
        path.quadraticCurveTo(width / 2, sourceY, width - 20, targetY)

        svg.append("path")
            .attr("d", path.toString())
            .attr("stroke", "#69b3a2")
            .attr("stroke-width", flowHeight)
            .attr("fill", "none")
            .attr("opacity", 0.6)
            .attr("stroke-linecap", "round")

        // Add flow value labels
        svg.append("text")
            .attr("x", width / 2)
            .attr("y", (sourceY + targetY) / 2)
            .attr("text-anchor", "middle")
            .attr("dominant-baseline", "middle")
            .attr("fill", "#333")
            .attr("font-size", "12px")
            .text(d.value)
    })

    // Add labels
    svg.append("text")
        .attr("x", 10)
        .attr("y", -5)
        .attr("text-anchor", "start")
        .attr("font-weight", "bold")
        .text("Device Type")

    svg.append("text")
        .attr("x", width - 10)
        .attr("y", -5)
        .attr("text-anchor", "end")
        .attr("font-weight", "bold")
        .text("Browser")

    // Add source labels
    svg.selectAll(".source-label")
        .data(sources)
        .enter()
        .append("text")
        .attr("class", "source-label")
        .attr("x", 25)
        .attr("y", d => sourceScale(d) + sourceScale.bandwidth() / 2 + 4)
        .attr("text-anchor", "start")
        .attr("font-size", "12px")
        .text(d)

    // Add target labels
    svg.selectAll(".target-label")
        .data(targets)
        .enter()
        .append("text")
        .attr("class", "target-label")
        .attr("x", width - 25)
        .attr("y", d => targetScale(d) + targetScale.bandwidth() / 2 + 4)
        .attr("text-anchor", "end")
        .attr("font-size", "12px")
        .text(d)
}
</script>

<style scoped>
/* Add some basic styling */
.alluvial-container {
    background: #f9f9f9;
    border-radius: 8px;
    padding: 20px;
}
</style>
