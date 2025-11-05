---
title: Chart
toc: false
---

# SNAP Benefits from 1988 to 2025
## A month breakdown of the benefits broken down by state/territory.


```js
import * as d3 from "npm:d3@7";
import * as Inputs from "npm:@observablehq/inputs@0.12";
import { sliderBottom } from "npm:d3-simple-slider@1";

const input = await FileAttachment("data/CleanSnapData.json").json();
const stateAbbrev = new Map([
  ["Alabama", "AL"], ["Alaska", "AK"], ["Arizona", "AZ"], ["Arkansas", "AR"],
  ["California", "CA"], ["Colorado", "CO"], ["Connecticut", "CT"], ["Delaware", "DE"],
  ["Florida", "FL"], ["Georgia", "GA"], ["Hawaii", "HI"], ["Idaho", "ID"],
  ["Illinois", "IL"], ["Indiana", "IN"], ["Iowa", "IA"], ["Kansas", "KS"],
  ["Kentucky", "KY"], ["Louisiana", "LA"], ["Maine", "ME"], ["Maryland", "MD"],
  ["Massachusetts", "MA"], ["Michigan", "MI"], ["Minnesota", "MN"], ["Mississippi", "MS"],
  ["Missouri", "MO"], ["Montana", "MT"], ["Nebraska", "NE"], ["Nevada", "NV"],
  ["New Hampshire", "NH"], ["New Jersey", "NJ"], ["New Mexico", "NM"],
  ["New York", "NY"], ["North Carolina", "NC"], ["North Dakota", "ND"],
  ["Ohio", "OH"], ["Oklahoma", "OK"], ["Oregon", "OR"], ["Pennsylvania", "PA"],
  ["Rhode Island", "RI"], ["South Carolina", "SC"], ["South Dakota", "SD"],
  ["Tennessee", "TN"], ["Texas", "TX"], ["Utah", "UT"], ["Vermont", "VT"],
  ["Virginia", "VA"], ["Washington", "WA"], ["West Virginia", "WV"],
  ["Wisconsin", "WI"], ["Wyoming", "WY"],
  ["District of Columbia", "DC"], ["Guam", "GU"], ["Virgin Islands", "VI"]
]);

function mapData(line) {
    const date = new Date(line.date);
    const month = d3.timeFormat("%Y-%m")(date)
    return {
        id: `${line.state}.${month}`,
        state: line.state,
        month: month,
        value: line.total_cost
    };
}


// da chart
function bubbleChart(data, {
    // setting default settings unless i say so
    width = 400,
    height = width / 2,
    padding = 3,
    color = null,
    colorAccessor = i => i.avg_per_person,
    rAccessor = i => i.value,
    rScale = globalRScale
    } = {}
    ) {


    const svg = d3.create("svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height]);

    if (!color) {
    const values = data.map(colorAccessor).filter(v => !isNaN(v));
    const [min, max] = d3.extent(values);
    color = d3.scaleSequential()
      .domain([min, max])
      .interpolator(d3.interpolateYlGnBu);
  }

  const nodes = (data
    .map((d) => {
        const lay = layoutByState.get(d.state); // precomputed in your baseline step
        if (!lay) return null; // skip anything we can't place
        return {
          state: d.state,
          value: rAccessor(d),
          avg_per_person: d.avg_per_person,
          x: lay.xN * width,     // normalized → pixel coords
          y: lay.yN * height,    // normalized → pixel coords
          r: rScale(rAccessor(d))// size from global scale (consistent across months)
        };
      })
      .filter(Boolean));

    // da circles
    svg.selectAll("circle")
    .data(nodes)
    .join("circle")
    .attr("cx", i => i.x)
    .attr("cy", i => i.y)
    .attr("r", i => i.r)
    .attr("stroke", "black")
    .attr("fill", (i) =>
      color ? color(colorAccessor({ avg_per_person: i.avg_per_person })) : "none"
    )

    // state labels for da circles
    svg.selectAll("text")
        .data(nodes.filter(d => d.r > 10))
        .join("text")
        .attr("x", d => d.x)
        .attr("y", d => d.y)
        .attr("text-anchor", "middle")
        .attr("dominant-baseline", "middle")
        .attr("font-size", d => Math.max(8, d.r / 3))
        .attr("fill", "black")
        .text(i => stateAbbrev.get(i.state));
    
    // color legend 
    const legendWidth = 250;
    const legendHeight = 10;
    const marginRight = 12;
    const marginTop = 12;

    const legend = svg.append("g")
    .attr("transform", `translate(${width - legendWidth - marginRight}, ${marginTop})`);

    const defs = svg.append("defs");
    const gradId = "color-legend";
    const [minVal, maxVal] = color.domain();

    const gradient = defs.append("linearGradient")
        .attr("id", gradId)
        .attr("x1", "0%").attr("x2", "100%");

    const steps = 60;
    for (let i = 0; i < steps; i++) {
        const t = i / (steps - 1);
        const v = minVal + t * (maxVal - minVal);
        gradient.append("stop")
            .attr("offset", `${t * 100}%`)
            .attr("stop-color", color(v));
    }

    legend.append("rect")
        .attr("width", legendWidth)
        .attr("height", legendHeight)
        .attr("fill", `url(#${gradId})`)
        .attr("stroke", "#ccc");
    
    const legendScale = d3.scaleLinear()
        .domain([minVal, maxVal])
        .range([0, legendWidth]);

    const legendAxis = d3.axisBottom(legendScale)
        .ticks(4)
        .tickFormat(d3.format("$,.0f"));

    legend.append("g")
        .attr("transform", `translate(0,${legendHeight})`)
        .call(legendAxis)
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll("text").attr("font-size", 10));
    
    // total cost at bot of screen
    const total = d3.sum(data, d => d.value);

    svg.append("text")
        .attr("x", width - 8)
        .attr("y", height - 8)
        .attr("text-anchor", "end")
        .attr("font-size", 30)
        .attr("font-weight", "bold")
        .attr("fill", "black")
        .attr("paint-order", "stroke")
        .attr("stroke", "white")
        .attr("stroke-width", 1.5)
        .text(`Total: ${d3.format("$,.0f")(total)}`);

    return svg.node();
}

function stateData(month) {
    return input
        .filter(i => i.state !== "US Summary")
        .filter(i => d3.timeFormat("%Y-%m")(new Date(i.date)) === month)
        .map(i => ({ 
            state: i.state, 
            value: i.total_cost,
            avg_per_person: i.avg_per_person
        }));
}

function USData(month) {
    return (input
        .filter(i => i.state === "US Summary")
        .filter(i => d3.timeFormat("%Y-%m")(new Date(i.date)) === month)
        .map(i => ({state: i.state,value: +i.total_cost}))
        );
}



const format = d3.timeFormat("%Y-%m");

// get the earliest month in your data
const firstMonth = format(new Date(d3.min(input, d => d.date)));
console.log("first month:", firstMonth);

const monthsSet = new Set();
for (const x of input){
    monthsSet.add(format(new Date(x.date)));
}
const months = Array.from(monthsSet).sort();
const month = months[0];


// get spots for the states
const byState = d3.rollup(
    input.filter(i => i.state !== "US Summary"),
    val => ({v0: d3.median(val, i => i.total_cost)}),
    i => i.state
);

// i dont want the circles moving all around
const baseline = Array.from(byState, ([state, o]) => ({ state, value: o.v0 }));

const LAYOUT_W = 1000, LAYOUT_H = 1000;
const baselineRoot = d3.pack()
  .size([LAYOUT_W, LAYOUT_H])
  .padding(3)(d3.hierarchy({children: baseline}).sum(d => d.value));

const layoutByState = new Map();
for (const leaf of baselineRoot.leaves()) {
  const s = leaf.data.state;
  const xN = leaf.x / LAYOUT_W;
  const yN = leaf.y / LAYOUT_H;
  const r0 = leaf.r;
  const v0 = byState.get(s).v0
  layoutByState.set(s, { xN, yN, r0, v0 });
}

// global scale for all data to make circles grow over time
const globalRScale = d3.scaleSqrt()
  .domain([0, d3.max(input.filter(i => i.state !== "US Summary"), d => d.total_cost)])
  // size of the circles
  .range([2, 150]);

// color scale but for all months
const globalColor = d3.scaleSequential(d3.interpolateTurbo)
  .domain([1, 500])
  .clamp(true);

```

```js

// get all states for that month
const statesMonth = stateData(firstMonth);
console.log("statesMonth:", statesMonth.length, statesMonth.slice(0, 5));

// get the US summary row(s) for that month
const usMonth = USData(firstMonth);
console.log("usMonth:", usMonth);

// slider shit
const card = d3.select("#chart-card");

// render the chart
function render(selectedMonth) {
  card.selectAll("*").remove();
  const width = card.node().getBoundingClientRect().width;
  card.append(() => bubbleChart(stateData(selectedMonth), { width, color: globalColor }));
  d3.select("#month-label").text(selectedMonth);
}

const parseMonth = d3.timeParse("%Y-%m");
const formatMonth = d3.timeFormat("%Y-%m");

const monthDates = months.map(m => parseMonth(m)).sort(d3.ascending);

const slider = sliderBottom()
  .min(monthDates[0])
  .max(monthDates[monthDates.length - 1])
  .width(400)
  // show spaced labels
  .tickValues(monthDates.filter((d, i) => i % Math.max(1, Math.floor(monthDates.length / 6)) === 0))
  .tickFormat(formatMonth)
  .default(parseMonth(month))
  .fill("#85bb65")
  .on("onchange", (val) => {
    const nearest = d3.least(monthDates, d => Math.abs(d - val));
    render(formatMonth(nearest));
  });

const gSlider = d3
  .select("#slider_range")
  .append("svg")
  .attr("width", 520)
  .attr("height", 90)
  .append("g")
  .attr("transform", "translate(60,30)");

gSlider.call(slider);
render(month);

```


<div class="grid grid-cols-1">
  <div id="chart-card" class="card"></div>
</div>

### Month: **<span id="month-label"></span>**
<div id="slider_range"></div>

