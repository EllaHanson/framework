---
title: Chart
toc: false
---

# Howdy


```js
import * as d3 from "npm:d3@7";
import * as Inputs from "npm:@observablehq/inputs@0.12";

const input = await FileAttachment("data/CleanSnapData.json").json();

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

function bubbleChart(data, options = {}) {
    const width  = options.width  ?? 400;
    const height = options.height ?? width;
    const padding = options.padding ?? 3;

    // create pack layout and compute heirarchy
    const root = (d3.pack()
        .size([width,height])
        .padding(padding)(
            d3.hierarchy({children: data}).sum(i => i.value)
        ))

    const svg = d3.create("svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height]);

    svg.selectAll("circle")
    .data(root.leaves())
    .join("circle")
    .attr("cx", i => i.x)
    .attr("cy", i => i.y)
    .attr("r", i => i.r)
    .attr("stroke", "black")
    .attr("fill", "none");



    return svg.node();
}

function stateData(month) {
    return input
        .filter(i => i.state !== "US Summary")
        .filter(i => d3.timeFormat("%Y-%m")(new Date(i.date)) === month)
        .map(i => ({ state: i.state, value: i.total_cost }));
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
```

```js

// get all states for that month
const statesMonth = stateData(firstMonth);
console.log("statesMonth:", statesMonth.length, statesMonth.slice(0, 5));

// get the US summary row(s) for that month
const usMonth = USData(firstMonth);
console.log("usMonth:", usMonth);

```


<div class="grid grid-cols-1">
  <div class="card">${resize(width => bubbleChart(stateData(month), {width}))}</div>
</div>


### Month: **${month}**

