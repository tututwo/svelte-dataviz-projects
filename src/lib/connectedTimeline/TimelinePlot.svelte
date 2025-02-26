<script>
  import { line, curveCatmullRom } from "d3";
  import { cubicOut } from "svelte/easing";
  import { Tween } from "svelte/motion";
  import { fade } from "svelte/transition";

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
        currentPoints: isDistributed ? distributedPoints : centeredPoints,
        distributedPoints,
        centeredPoints,
        startColor: getColor(yScale(bar[yKey])),
        endColor: getColor(yScale(nextBar[yKey])),
      };
    })
  );
  // Create tweens for each connection
  let connectionTweens = $state([]);

  // Initialize tweens when connections change
  $effect(() => {
    // Create a tween for each connection if it doesn't exist
    if (connectionTweens.length !== connections.length) {
      connectionTweens = connections.map((conn) => {
        return new Tween(conn.currentPoints, {
          duration: 800,
          easing: cubicOut,
        });
      });
    }
  });

  // Update tweens when isDistributed changes
  $effect(() => {
    // Update each tween's target
    connectionTweens.forEach((tween, i) => {
      const conn = connections[i];
      const targetPoints = isDistributed
        ? conn.distributedPoints
        : conn.centeredPoints;

      // Normalize points if needed
      if (targetPoints.length !== tween.current.length) {
        const [normalizedCurrent, normalizedTarget] = normalizePoints(
          tween.current,
          targetPoints
        );
        // Set the tween with normalized points
        tween.set(normalizedTarget);
      } else {
        // Set the tween directly
        tween.set(targetPoints);
      }
    });
  });
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

  // Function to normalize points arrays to have the same length
  function normalizePoints(pointsA, pointsB) {
    // If arrays already have the same length, return them as is
    if (pointsA.length === pointsB.length) {
      return [pointsA, pointsB];
    }

    // Simple approach - pad the shorter array with interpolated points
    if (pointsA.length < pointsB.length) {
      const newPoints = [...pointsA];
      const diff = pointsB.length - pointsA.length;
      const lastIndex = pointsA.length - 1;

      for (let i = 0; i < diff; i++) {
        // Add points that are interpolated between the last two existing points
        const ratio = (i + 1) / (diff + 1);
        newPoints.push({
          x:
            pointsA[lastIndex].x +
            ratio * (pointsA[lastIndex].x - pointsA[lastIndex - 1].x),
          y: pointsA[lastIndex].y,
        });
      }
      return [newPoints, pointsB];
    } else {
      // If pointsB is shorter, swap the arguments and reverse the result
      const [normalized, original] = normalizePoints(pointsB, pointsA);
      return [original, normalized];
    }
  }
</script>

<button onclick={handleTransition}>
  {isDistributed ? "Center" : "Distribute"}
</button>
<svg class="w-full h-full">
  {#if data.length > 0}
    <!-- Draw connecting curves first -->
    {#each connectionTweens as tween, i}
      <path
        d={connectionLine(tween.current)}
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
