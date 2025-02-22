<script>
  import { line, curveCatmullRom } from "d3";

  import { cubicOut } from "svelte/easing";
  let { data, xScale, yScale, xKey, yKey, widthKey, barHeight = 10 } = $props();
  let isDistributed = $state(false);

  let connectionLine = $derived(
    line()
      .x((d) => d.x)
      .y((d) => d.y)
      .curve(curveCatmullRom.alpha(0.3)) // This gives us a smoother curve
  );
  // Create derived connection paths between bars
  let connections = $derived(
    data.slice(0, -1).map((bar, i) => {
      const nextBar = data[i + 1];
      const centerY = yScale.range()[0] / 2;

      // Calculate both centered and distributed points
      const distributedPoints = [
        {
          x: xScale(bar[xKey]) + xScale(bar[widthKey]) - 2,
          y: yScale(bar[yKey]) + barHeight / 2,
        },
        {
          x: xScale(bar[xKey]) + xScale(bar[widthKey]),
          y: yScale(bar[yKey]) + barHeight / 2,
        },
        {
          x: xScale(nextBar[xKey]),
          y: yScale(nextBar[yKey]) + barHeight / 2,
        },
        {
          x: xScale(nextBar[xKey]),
          y: yScale(nextBar[yKey]) + barHeight / 2,
        },
      ];

      const centeredPoints = [
        {
          x: xScale(bar[xKey]) + xScale(bar[widthKey]) - 2,
          y: centerY + barHeight / 2,
        },
        {
          x: xScale(bar[xKey]) + xScale(bar[widthKey]),
          y: centerY + barHeight / 2,
        },
        {
          x: xScale(nextBar[xKey]),
          y: centerY + barHeight / 2,
        },
        {
          x: xScale(nextBar[xKey]),
          y: centerY + barHeight / 2,
        },
      ];

      return {
        points: isDistributed ? distributedPoints : centeredPoints,
        startColor: getColor(yScale(bar[yKey])),
        endColor: getColor(yScale(nextBar[yKey])),
      };
    })
  );

  function getColor(scaledValue) {
    if (scaledValue < yScale.range()[0] / 3) return "#fd9226";
    if (scaledValue < (2 * yScale.range()[0]) / 3) return "#fa4f43";
    return "#a20f79";
  }
  function handleTransition() {
    isDistributed = !isDistributed;
  }

  function verticalMove(node, { from, to }) {
    const dy = from - to;

    return {
      duration: 800,
      easing: cubicOut,
      css: (t, u) => `transform: translateY(${u * dy}px)`,
    };
  }
 
</script>

<button onclick={handleTransition}>
  {isDistributed ? "Center" : "Distribute"}
</button>
<svg class="w-full h-full">
  {#if data.length > 0}
    <!-- Draw connecting curves first -->
    {#each connections as conn}
      <path
        d={connectionLine(conn.points)}
        fill="none"
        stroke="#666"
        stroke-width="1.5"
        opacity="0.5"
      />
    {/each}

    <!-- Draw the bars on top -->
    {#each data as shot (shot[xKey])}
      <!-- Use a unique key for proper transitions -->
      {#if isDistributed}
        <rect
          transition:verticalMove={{
            from: yScale.range()[0] / 2, // Start from center
            to: yScale(shot[yKey]), // Move to final position
          }}
          width={xScale(shot[widthKey]) - 2}
          height={barHeight}
          x={xScale(shot[xKey])}
          y={yScale(shot[yKey])}
          fill={getColor(yScale(shot[yKey]))}
        />
      {:else}
        <rect
          transition:verticalMove={{
            from: yScale(shot[yKey]), // Start from current position
            to: yScale.range()[0] / 2, // Move to center
          }}
          width={xScale(shot[widthKey]) - 2}
          height={barHeight}
          x={xScale(shot[xKey])}
          y={yScale.range()[0] / 2}
          fill={getColor(yScale(shot[yKey]))}
        />
      {/if}
    {/each}
  {/if}

  
</svg>


