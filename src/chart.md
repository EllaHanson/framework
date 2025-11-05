---
title: Chart
toc: false
---

# Howdy


```js
import * as d3 from "npm:d3@7";
import * as Inputs from "npm:@observablehq/inputs@0.12";

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

function bubbleChart(data, {
    width = 400,
    height = width,
    padding = 3,
    color = null,
    colorAccessor = i => i.avg_per_person
    } = {}
    ) {

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

    if (!color) {
    const values = data.map(colorAccessor).filter(v => !isNaN(v));
    const [min, max] = d3.extent(values);
    color = d3.scaleSequential()
      .domain([min, max])
      .interpolator(d3.interpolateYlGnBu);
  }

    svg.selectAll("circle")
    .data(root.leaves())
    .join("circle")
    .attr("cx", i => i.x)
    .attr("cy", i => i.y)
    .attr("r", i => i.r)
    .attr("stroke", "black")
    .attr("fill", i => color ? color(colorAccessor(i.data)) : "none");

    
    svg.selectAll("text")
        .data(root.leaves().filter(d => d.r > 10))
        .join("text")
        .attr("x", d => d.x)
        .attr("y", d => d.y)
        .attr("text-anchor", "middle")
        .attr("dominant-baseline", "middle")
        .attr("font-size", d => Math.max(8, d.r / 3))
        .attr("fill", "black")
        .text(i => stateAbbrev.get(i.data.state));

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

<div id="slider_range"></div>

<script src="https://unpkg.com/d3-simple-slider"></script>


### Month: **${month}**

