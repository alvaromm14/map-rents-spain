<script>
	import * as topojson from 'topojson-client';
	import { geoPath } from 'd3-geo';
  import { geoConicConformalSpain } from 'd3-composite-projections';
	import { draw } from 'svelte/transition';
  import municipalities from "$data/municipalities.json";
  import capitales from "$data/alquileres.js";
  import { scaleLinear } from 'd3-scale';
  import { max } from 'd3-array';
  import Tooltip from '$components/Tooltip.svelte';

  let provincias = topojson.feature(municipalities, municipalities.objects.provinces).features;
  let bordes = topojson.mesh(municipalities, municipalities.objects.autonomous_regions, (a, b) => a !== b);

  $: compositionBorders = projection.getCompositionBorders();

  let width = 400;
  $: height = width * 0.70;
  
  $: projection = geoConicConformalSpain().scale(width > 1200 ? 4000 : width * 3.5).translate(width > 1200 ? [width / 2, 400] : [width / 2, height / 2]);
  $: path = geoPath(projection);

  $: sizeScale = scaleLinear()
    .domain([0.2, max(capitales, d => d.Porcentaje)])
    .range(width > 850 ? [2.5, 40] : width > 610 ? [2, 30] : [2, 10]);

  const colorScale = scaleLinear()
    .domain([0.21, max(capitales, d => d.Porcentaje)])
    .range(["#FEDC33", "#B31168"])

  const colorBorderScale = scaleLinear()
    .domain([0.2, max(capitales, d => d.Porcentaje)])
    .range(["#CCA900", "#660037"])

  let selection;

  window.addEventListener("DOMContentLoaded", (event) => {
	function updateIframeHeight() {
		const el = document.documentElement;
		const rect = el.getBoundingClientRect();
		const styles = window.getComputedStyle(el);
		const margin = parseFloat(styles.marginTop) + parseFloat(styles.marginBottom);
		const height = Math.ceil(rect.height + margin);

		window.parent.postMessage({
			type: "resize-iframe",
			value: height
		}, "*");
	}
	updateIframeHeight();

	if (window.ResizeObserver) {
		new ResizeObserver(() => {
			updateIframeHeight();
		}).observe(document.documentElement);
	} else {
		window.addEventListener('load', updateIframeHeight);
		window.addEventListener('resize', updateIframeHeight);
	}

	window.addEventListener("message", (event) => {
		if (event.data.type === "request-resize") {
			updateIframeHeight();
		}
	}, false);
});

</script>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<div class="chart-container" bind:clientWidth={width}
on:click={() => {
  selection = null;
}}>
  <svg {width} {height} on:click|stopPropagation>
    {#each provincias as provincia}
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      <path d={path(provincia)} fill="#EBEBEB" stroke="black" stroke-width="0.05"/>
    {/each}

    <path d={path(bordes)} fill="none" stroke="black" stroke-width="0.1" />
    <path d={compositionBorders} fill="none" stroke="black" stroke-width="0.5"/>

    {#each capitales as capital}
    <!-- svelte-ignore a11y-no-noninteractive-tabindex -->
    <!-- svelte-ignore a11y-mouse-events-have-key-events -->
    <circle
    cx={projection([capital.Longitude, capital.Latitude])[0]}
    cy={projection([capital.Longitude, capital.Latitude])[1]}
    r={sizeScale(capital.Porcentaje)}
    fill={colorScale(capital.Porcentaje)}
    fill-opacity={ selection ? selection === capital ? "100%" : "30%" : "100%"}
    stroke={colorBorderScale(capital.Porcentaje)}
    stroke-width="0.8"
    stroke-opacity={ selection ? selection === capital ? "100%" : "30%" : "100%"}
    on:mouseover={() => {
      selection = capital;
    }}
    on:focus={() => {
      selection = capital;
    }}
    tabindex="0"
    on:mouseout={() => {
      selection = null;
    }}
    />
    {/each}
  </svg>

  {#if selection}
    <Tooltip {projection} {selection} {width} {height} {colorScale}/>
  {/if}

</div>


<style>
  .chart-container {
    max-height: 810px;
  }

  svg {
    max-height: 810px;
  }

</style>