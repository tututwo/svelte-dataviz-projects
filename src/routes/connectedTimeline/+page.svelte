<script>
  // IMPORTANT: components
  import TimelinePlot from "$lib/connectedTimeline/TimelinePlot.svelte";

  // Utility Modules
  import { onMount } from "svelte";

  //   External libraries
  import * as d3 from "d3";
  // Create a proper type for your data
  /** @type {{ 
    trailer?: { 
      trailer_length_ms: number;
      shots: Array<{ start_ms: number; length_ms?: number }> 
    } 
  }} */
  let data = $state({}); // Keep empty object as default to avoid undefined errors

  const xKey = "start_ms";
  const yKey = "match_ms";
  const widthKey = "length_ms";

  let figureWidth = $state(0);
  let figureHeight = $state(0);

  onMount(async () => {
    const response = await fetch(
      "https://static01.nyt.com/newsgraphics/2013/01/04/movie-marker/b1a9c2fc1689fd89ecff4a7d2be5e5757716e344/movies.json"
    );
    const dataJSON = await response.json();

    // Create a new object instead of mutating existing data
    const movieData = dataJSON.filter(
      (/** @type {{ intro: string; }} */ d) =>
        d.intro ===
        "“Silver Linings Playbook” follows the standard model for trailers, according to <a href='http://www.billwoolery.com/'>Bill Woolery</a>, a trailer specialist in Los Angeles who once worked on trailers for movies like “The Usual Suspects” and “E.T. the Extra-Terrestrial.” While introducing the movie’s story and its characters, the trailer largely follows the order of the film itself."
    )[0];
    let firstButEndingDuration = movieData.trailer.trailer_length_ms;
    // Process shots first
    const shots = [...movieData.trailer.shots].reverse().map((shot) => {
      return {
        ...shot,
        length_ms:
          firstButEndingDuration - (firstButEndingDuration = shot.start_ms),
      };
    });

    // Then assign everything at once to data
    data = {
      ...movieData,
      trailer: {
        ...movieData.trailer,
        shots,
      },
    };
  });

  let shots = $derived(data.trailer?.shots ?? []);
  let xDomain = $derived(
    shots?.length ? [0, d3.max(shots, (d) => d[xKey])] : [0, 120000]
  );

  let yDomain = $derived(
    shots?.length ? d3.extent(shots, (d) => d[yKey]).reverse() : [0, 120000]
  );
  let xScale = $derived(
    d3.scaleLinear().domain(xDomain).range([0, figureWidth])
  );
  let yScale = $derived(
    d3.scaleLinear().domain(yDomain).range([figureHeight, 0])
  );
</script>

<article class="max-w-7xl mx-auto px-4 py-8">
  <header class="mb-12">
    <h1 class="text-4xl font-serif font-bold mb-4">
      Dissecting a Trailer: The Parts of the Film That Make the Cut
    </h1>
    <p class="text-xl text-muted-foreground">
      How scenes from five of the nine best picture nominees were reassembled to
      promote the films.
    </p>
  </header>

  <div class="grid lg:grid-cols-3 gap-8 mb-12">
    <div class="lg:col-span-2">
      <section>
        <h2 class="text-2xl font-bold mb-4">Silver Linings Playbook</h2>
        <p class="text-gray-700 mb-6">
          "Silver Linings Playbook" follows the standard model for trailers,
          according to Bill Woolery, a trailer specialist in Los Angeles who
          once worked on trailers for movies like "The Usual Suspects" and "E.T.
          the Extra-Terrestrial." While introducing the movie's story and its
          characters, the trailer largely follows the order of the film itself.
        </p>
      </section>
    </div>

    <div class="lg:col-span-1">
      <div class="relative aspect-video bg-black rounded-lg overflow-hidden">
        <button
          class="absolute inset-0 flex items-center justify-center bg-black/30 hover:bg-black/40 transition-colors"
        >
        </button>
      </div>
    </div>
  </div>

  <secion class="mt-8">
    <div class="relative overflow-x-auto">
      <div class="min-w-[800px]">
        <div class="flex justify-between mb-2 text-sm text-gray-600">
          <span>Start of trailer →</span>
          <div class="flex justify-between flex-1 px-4">
            <span>0 sec</span>
            <span>30 sec</span>
            <span>60 sec</span>
            <span>90 sec</span>
            <span>120 sec</span>
          </div>
        </div>

        <figure
          bind:clientWidth={figureWidth}
          bind:clientHeight={figureHeight}
          class="h-[300px]"
        >
          <TimelinePlot
            data={d3.sort(shots, d => d[xKey])}
            {xScale}
            {yScale}
            {xKey}
            {yKey}
            {widthKey}
          />
        </figure>

        <div class="grid grid-cols-3 gap-8 mt-6 text-sm text-gray-600">
          <div>
            The trailer's opening shot — an image of the family's home — appears
            near the end of the film, but there are similar shots near the
            beginning of the movie.
          </div>
          <div>
            A handful of very short shots are never seen in the film, although
            most are shown from alternate camera angles.
          </div>
          <div>
            Shots that accompany the main actors' names are also shown out of
            order.
          </div>
        </div>
      </div>
    </div>
  </secion>
</article>
