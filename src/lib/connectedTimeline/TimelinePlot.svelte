<!-- @ts-nocheck -->
<script>
  import { line, curveCatmullRom } from "d3";
  import { cubicOut } from "svelte/easing";
  import { Tween } from "svelte/motion";

  let { data, xScale, yScale, xKey, yKey, widthKey, barHeight = 10 } = $props();
  let isDistributed = $state(false);
  
  // Animation duration for all animations
  const ANIMATION_DURATION = 800;
  
  /**
   * This component uses Svelte 5's Tween class for animations:
   * 1. Path morphing: Animates the connecting paths between bars when toggling between distributed and centered views
   * 2. Rectangle positioning: Animates the vertical movement of rectangles
   * 
   * The Tween approach is preferred over transitions in Svelte 5 for this use case because:
   * - It provides more control over the animation
   * - It works well for elements that remain in the DOM (not being added/removed)
   * - It allows for custom interpolation between different data structures (like points arrays)
   */

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
  
  // Create tweens for rectangle positions
  let rectPositionTweens = $state([]);

  // Instead of creating new Tween instances every time, reuse existing ones
$effect(() => {
  // Adjust the length of the connectionTweens array if needed
  if (connectionTweens.length > connections.length) {
    // Remove excess tweens
    connectionTweens = connectionTweens.slice(0, connections.length);
  } else if (connectionTweens.length < connections.length) {
    // Add new tweens for new connections
    const newTweens = connections.slice(connectionTweens.length).map(conn => {
      return new Tween(conn.currentPoints, {
        duration: ANIMATION_DURATION,
        easing: cubicOut,
      });
    });
    connectionTweens = [...connectionTweens, ...newTweens];
  }
  
  // Update existing tweens with new values
  connections.forEach((conn, i) => {
    if (connectionTweens[i]) {
      connectionTweens[i].set(isDistributed ? conn.distributedPoints : conn.centeredPoints);
    }
  });
});

// Similar approach for rectPositionTweens
$effect(() => {
  // Adjust the length of the rectPositionTweens array if needed
  if (rectPositionTweens.length > data.length) {
    // Remove excess tweens
    rectPositionTweens = rectPositionTweens.slice(0, data.length);
  } else if (rectPositionTweens.length < data.length) {
    // Add new tweens for new data points
    const centerY = yScale.range()[0] / 2;
    const newTweens = data.slice(rectPositionTweens.length).map(shot => {
      const distributedY = yScale(shot[yKey]);
      return new Tween(
        isDistributed ? distributedY : centerY, 
        {
          duration: ANIMATION_DURATION,
          easing: cubicOut,
        }
      );
    });
    rectPositionTweens = [...rectPositionTweens, ...newTweens];
  }
});

  // Update connection tweens when isDistributed changes
  $effect(() => {
    // Update each connection tween's target
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
    
    // Update each rectangle position tween
    rectPositionTweens.forEach((tween, i) => {
      const shot = data[i];
      const centerY = yScale.range()[0] / 2;
      const distributedY = yScale(shot[yKey]);
      
      tween.set(isDistributed ? distributedY : centerY);
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

<button
  onclick={handleTransition}
  type="button"
  class="rounded-md bg-white px-3.5 py-2.5 text-sm font-semibold text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 hover:bg-gray-50"
>
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

    <!-- Draw the bars on top using tweened positions -->
    {#each data as shot, i (shot[xKey])}
      {#if rectPositionTweens[i]}
        <rect
          width={xScale(shot[widthKey]) - 2}
          height={barHeight}
          x={xScale(shot[xKey])}
          y={rectPositionTweens[i].current}
          fill={getColor(yScale(shot[yKey]))}
        />
      {/if}
    {/each}
  {/if}
</svg>
