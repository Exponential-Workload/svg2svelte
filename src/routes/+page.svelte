<script lang="ts">
  import CodeMirror from '$lib/CodeMirror.svelte';
  import Meta from '../lib/Meta/Meta.svelte';
  import svgo from 'svgo';
  import xmlFormat from 'xml-formatter';
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
            multipass: true,
            plugins: [
              {
                name: 'preset-default',
                params: {
                  overrides: {
                    removeViewBox: false,
                  },
                },
              },
              'collapseGroups',
              'convertEllipseToCircle',
              'convertPathData',
              'removeDimensions',
            ],
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
        const xml = `${svg.documentElement.outerHTML}`;
        svelte =
          '<' +
          `script lang='ts'>
  export let width = ${width};
  export let height = ${
    width === height ? 'width' : `width / ${width} * ${height}`
  };${Object.entries(colourVariables)
            .map(
              ([key, value]) => `
  export let ${value} = '${key}';`
            )
            .join('')}
</` +
          `script>
${xmlFormat(xml, {
  indentation: '  ',
})}`;
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
    if (Math.abs(val.length - input.length) > 64 && !val.includes('<script')) {
      try {
        // parse
        const parser = new DOMParser();
        parser.parseFromString(val, 'image/svg+xml');
        // parse success
        input = val;
      } catch (error) {}
    }
  };
  let copied = false;
  let saved = false;
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
      saved = true;
      setTimeout(() => {
        saved = false;
      }, 2500);
    } else if (e.ctrlKey && e.key.toLowerCase() === 'e') {
      e.preventDefault();
      const isLower = e.key.toLowerCase() === e.key;
      try {
        navigator.clipboard.writeText(isLower ? svelte : svgoOut);
        copied = true;
        setTimeout(() => {
          copied = false;
        }, 2500);
      } catch (error) {
        alert(error);
      }
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
  <table
    style="font-weight:400;font-family:Inter, system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;position:fixed;bottom:1ch;left:50vw;transform:translate(-50%,0);background:#fff2;backdrop-filter:blur(16px);border-radius:8px;padding:1ch;transition:1s;"
  >
    <tbody>
      <tr>
        <th style="text-align:left;">Shortcut</th>
        <th style="text-align:left;">Action</th>
      </tr>
      <tr>
        <td>Ctrl+S </td>
        <td>Save Svelte </td>
      </tr>
      <tr>
        <td style="padding-right:15px;">Ctrl+Shift+S </td>
        <td>Save SVGO SVG</td>
      </tr>
      <tr>
        <td>Ctrl+E </td>
        <td>Copy Svelte</td>
      </tr>
      <tr>
        <td>Ctrl+Shift+E </td>
        <td>Copy SVGO SVG</td>
      </tr>
    </tbody>
  </table>
  <!-- toasts -->
  {#if copied || saved}
    <div
      style="position:fixed;bottom:0.5rem;left:50vw;transform:translate(-50%,0);background:#fff2;backdrop-filter:blur(16px);border-radius:8px;padding:1ch;transition:1s;text-align:center;width:10vw;"
    >
      <p>{copied ? 'Copied' : 'Saved'}!</p>
    </div>
  {/if}
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
