<script>
  import { onMount, onDestroy } from "svelte";
  import { fade } from "svelte/transition";
  import * as d3 from "d3";
  import randomData from "../assets/random_data.json";
  import irisData from "../assets/iris_data.json";
  import verticalData from "../assets/vertical_data.json";
  import spiralData from "../assets/spiral_data.json";

  let color_1 = "#4A90E2";
  let color_2 = "#8BC34A";
  let color_3 = "#FF5733";

  export let index;
  let svg;
  let width, height;
  let xScale, yScale;
  let xAxis, yAxis;
  let margin = { top: 20, right: 20, bottom: 40, left: 50 };
  const padding = 0.25;

  let points = [];
  let velocities = [];
  let animationFrameId;
  let initialLoad = true;

  const randomDomain = { x: [0, 1], y: [0, 1] };
  const irisDomain = calculateDomain(irisData);
  const spiralDomain = calculateDomain(spiralData);
  const verticalDomain = calculateDomain(verticalData);

  function calculateDomain(data) {
    const xExtent = d3.extent(data, (d) => d.x);
    const yExtent = d3.extent(data, (d) => d.y);
    const xPadding = (xExtent[1] - xExtent[0]) * padding;
    const yPadding = (yExtent[1] - yExtent[0]) * padding;
    return {
      x: [xExtent[0] - xPadding, xExtent[1] + xPadding],
      y: [yExtent[0] - yPadding, yExtent[1] + yPadding],
    };
  }

  function initializePoints() {
    points = randomData.map((d, i) => ({
      id: i,
      x: d.x,
      y: d.y,
      color: d.class === 0 ? color_1 : color_2,
    }));
    velocities = points.map(() => ({
      vx: (Math.random() - 0.5) * 0.005,
      vy: (Math.random() - 0.5) * 0.005,
    }));
  }

  function startAnimation() {
    function update() {
      points.forEach((point, index) => {
        point.x += velocities[index].vx;
        point.y += velocities[index].vy;

        if (point.x > xScale.domain()[1] || point.x < xScale.domain()[0])
          velocities[index].vx *= -1;
        if (point.y > yScale.domain()[1] || point.y < yScale.domain()[0])
          velocities[index].vy *= -1;
      });

      const selection = d3
        .select(svg)
        .select(".points")
        .selectAll("circle")
        .data(points, (d) => d.id);

      selection.join(
        (enter) =>
          enter
            .append("circle")
            .attr("cx", (d) => xScale(d.x))
            .attr("cy", (d) => yScale(d.y))
            .attr("r", 5)
            .attr("fill", (d) => d.color),
        (update) =>
          update.attr("cx", (d) => xScale(d.x)).attr("cy", (d) => yScale(d.y)),
        (exit) => exit.remove()
      );

      animationFrameId = requestAnimationFrame(update);
    }

    update();
  }

  function createScales() {
    xScale = d3.scaleLinear().range([0, width]);
    yScale = d3.scaleLinear().range([height, 0]);
  }

  function createAxes() {
    xAxis = d3.axisBottom(xScale).ticks(Math.max(2, Math.floor(width / 50)));
    yAxis = d3.axisLeft(yScale).ticks(Math.max(2, Math.floor(height / 50)));
  }

  function renderAxes() {
    d3.select(svg)
      .select(".x-axis")
      .attr("transform", `translate(${margin.left}, ${height + margin.top})`)
      .transition()
      .duration(500)
      .call(xAxis);

    d3.select(svg)
      .select(".y-axis")
      .attr("transform", `translate(${margin.left}, ${margin.top})`)
      .transition()
      .duration(500)
      .call(yAxis);
  }

  function renderPoints() {
    const selection = d3
      .select(svg)
      .select(".points")
      .selectAll("circle")
      .data(points, (d) => d.id);

    selection.join(
      (enter) =>
        enter
          .append("circle")
          .attr("cx", (d) => xScale(d.x))
          .attr("cy", (d) => yScale(d.y))
          .attr("r", 5)
          .attr("fill", (d) => d.color),
      (update) =>
        update
          .transition()
          .duration(1000)
          .attr("cx", (d) => xScale(d.x))
          .attr("cy", (d) => yScale(d.y)),
      (exit) => exit.remove()
    );
  }

  function updateScales(domain) {
    xScale.domain(domain.x);
    yScale.domain(domain.y);
    createAxes();
    renderAxes();
    renderPoints();
  }

  function resize() {
    if (index === 0) {
      margin = { top: 0, right: 0, bottom: 0, left: 0 };
      height = svg.clientHeight - margin.top - margin.bottom;
      width = svg.clientWidth - margin.left - margin.right;
    } else {
      margin = { top: 20, right: 20, bottom: 40, left: 50 };
      width = height = svg.clientHeight - margin.top - margin.bottom;
      //svg.clientWidth - margin.left - margin.right;
    }

    createScales();
    updateScales(getDomain());
  }

  onMount(() => {
    const observer = new ResizeObserver(resize);
    observer.observe(svg);

    height = svg.clientHeight - margin.top - margin.bottom;
    width = svg.clientWidth - margin.left - margin.right;
    createScales();
    createAxes();
    updateScales(getDomain());

    initializePoints();
    startAnimation();

    onDestroy(() => {
      observer.disconnect();
      cancelAnimationFrame(animationFrameId);
    });
  });

  function handleIndexChange(index) {
    cancelAnimationFrame(animationFrameId);
    switch (index) {
      case 0:
        initializePoints();
        updateScales(randomDomain);
        if (!initialLoad) {
          setTimeout(() => {
            startAnimation();
          }, 1000); // 1000ms delay before starting the animation
        } else {
          startAnimation();
          initialLoad = false;
        }
        break;
      case 1:
        points = irisData.map((d, i) => ({
          id: i,
          x: d.x,
          y: d.y,
          color:
            d.class === "Iris-setosa"
              ? color_1
              : d.class === "Iris-versicolor"
                ? color_2
                : color_3,
        }));
        updateScales(irisDomain);
        break;
      case 2:
        // Line of Best Fit Phase
        points = irisData.map((d, i) => ({
          id: i,
          x: d.x,
          y: d.y,
          color:
            d.class === "Iris-setosa"
              ? color_1
              : d.class === "Iris-versicolor"
                ? color_2
                : color_3,
        }));
        updateScales(irisDomain);
        break;
      case 3:
        points = verticalData.map((d, i) => ({
          id: i,
          x: d.x,
          y: d.y,
          color: d.class === 0 ? color_1 : color_2,
        }));
        updateScales(verticalDomain);
        break;
      case 4:
        points = spiralData.map((d, i) => ({
          id: i,
          x: d.x,
          y: d.y,
          color: d.class === 0 ? color_1 : color_2,
        }));
        updateScales(spiralDomain);
        break;
    }
  }

  function getDomain() {
    switch (index) {
      case 1:
      case 2: // Line of Best Fit Phase
        return irisDomain;
      case 3:
        return verticalDomain;
      case 4:
      case 5:
      case 6:
        return spiralDomain;
      default:
        return randomDomain;
    }
  }

  $: handleIndexChange(index);
  $: if (index >= 0) resize();
  $: svgStyle =
    index === 0
      ? "width: 100%; height: 100vh;"
      : "width: 100%; height: 100%; aspect-ratio: 1 / 1;";
  $: showAxes = index > 0;
</script>

<svg bind:this={svg} style={svgStyle}>
  <g
    class="x-axis"
    style="opacity: {showAxes ? 1 : 0};"
    in:fade={{ delay: 300, duration: 300 }}
  ></g>
  <g
    class="y-axis"
    style="opacity: {showAxes ? 1 : 0};"
    in:fade={{ delay: 300, duration: 300 }}
  ></g>
  <g
    class="points"
    transform={`translate(${showAxes ? margin.left : 0}, ${showAxes ? margin.top : 0})`}
  ></g>
</svg>