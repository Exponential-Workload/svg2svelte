<script lang="ts">
  import CodeMirror from '$lib/CodeMirror.svelte';
  import Meta from '../lib/Meta/Meta.svelte';
  import svgo from 'svgo';
  let input = `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1877 512">
  <!-- your svg here -->
</svg>`;
  let svelte = '';
  let svgoOut = '';
  $: {
    if (typeof DOMParser !== 'undefined') {
      try {
        const parser = new DOMParser();
        const svgout = svgo.optimize(
          input ||
            `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1877 512">
  <!-- your svg here -->
</svg>`,
          {
            floatPrecision: 3,
          }
        ).data;
        const svg = parser.parseFromString(svgout, 'image/svg+xml');
        const pohtml = svg.documentElement.outerHTML;
        if (
          pohtml.includes(
            'http://www.mozilla.org/newlayout/xml/parsererror.xml'
          )
        )
          throw new Error(`Failed to Parse:
${pohtml}`);
        svgoOut = svg.documentElement.outerHTML;
        // get width from either width or viewbox
        const width =
          svg.documentElement.getAttribute('width') ||
          svg.documentElement.getAttribute('viewBox')?.split(' ')[2];
        // get height from either height or viewbox
        const height =
          svg.documentElement.getAttribute('height') ||
          svg.documentElement.getAttribute('viewBox')?.split(' ')[3];
        svg.documentElement.setAttribute('width', `{width}px`);
        svg.documentElement.setAttribute('height', `{height}px`);
        // get all colours, and replace them with svelte variables
        const colours =
          svg.documentElement.querySelectorAll('*[fill], *[stroke]');
        let colourVariables: Record<string, string> = {};
        colours.forEach((colour) => {
          const colourValue =
            colour.getAttribute('fill') || colour.getAttribute('stroke');
          if (colourValue) {
            if (!colourVariables[colourValue]) {
              const idx = Object.keys(colourVariables).length;
              colourVariables[colourValue] = `colour${
                idx === 0 ? '' : idx + 1
              }`;
            }
            colour.setAttribute(
              colour.getAttribute('fill') ? 'fill' : 'stroke',
              `{${colourVariables[colourValue]}}`
            );
          }
        });
        svelte =
          '<' +
          `script lang='ts'>
  export let width = ${width};
  export let height = width / ${width} * ${height};${Object.entries(
            colourVariables
          )
            .map(
              ([key, value]) => `
  export let ${value} = '${key}';`
            )
            .join('')}
</` +
          `script>
${svg.documentElement.outerHTML}`;
      } catch (error) {
        console.warn(`Error messing with SVG:`, error);
        svelte = `<!--
${error}
-->`;
      }
    }
  }
  const readonlyChangeAttempt = ({ detail }: { detail: { value: string } }) => {
    const val = detail.value;
    if (Math.abs(val.length - input.length) > 64) {
      try {
        // parse
        const parser = new DOMParser();
        parser.parseFromString(val, 'image/svg+xml');
        // parse success
        input = val;
      } catch (error) {}
    }
  };
</script>

<svelte:window
  on:keydown={(e) => {
    if (e.ctrlKey && e.key.toLowerCase() === 's') {
      e.preventDefault();
      const a = document.createElement('a');
      const isLower = e.key.toLowerCase() === e.key;
      a.href =
        'data:text/plain;charset=utf-8,' +
        encodeURIComponent(isLower ? svelte : svgoOut);
      a.download = isLower ? 'svg.svelte' : 'svg.svg';
      a.click();
    }
  }}
/>

<svelte:head>
  <Meta
    tags={{
      title: 'svg2svelte',
      description:
        'Takes an svg, passes it through svgo, and turns it into a svelte component',
    }}
  />
</svelte:head>

<main>
  <div>
    <h1>svg2svelte</h1>
    <div style="max-height: 50vh;overflow-y:auto;">
      <p>SVG:</p>
      <CodeMirror bind:value={input} html />
    </div>
    <hr />
    <p>Svelte:</p>
    <div style="max-height: 50vh;overflow-y:auto;filter:grayscale(0.3)">
      <CodeMirror
        bind:value={svelte}
        html
        readonly
        on:readOnlyChangeAttempt={readonlyChangeAttempt}
      />
    </div>
    <p>SVGO:</p>
    <div style="max-height: 50vh;overflow-y:auto;filter:grayscale(0.3)">
      <CodeMirror
        bind:value={svgoOut}
        html
        readonly
        on:readOnlyChangeAttempt={readonlyChangeAttempt}
      />
    </div>
  </div>
</main>

<style lang="scss">
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    width: 100vw;
    > div {
      width: calc(100vw - 60px);
    }
  }
</style>
