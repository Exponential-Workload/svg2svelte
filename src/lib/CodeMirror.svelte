<script lang="ts">
  import { EditorView, basicSetup } from 'codemirror';
  import { css as cssLang } from '@codemirror/lang-css';
  import { html as htmlLang } from '@codemirror/lang-html';
  import { oneDark } from '@codemirror/theme-one-dark';
  import { createEventDispatcher, onMount } from 'svelte';

  let element: HTMLDivElement;
  export let value = ``;
  export let html = false;
  export let readonly = false;
  let editor: EditorView;
  // event
  const dispatch = createEventDispatcher<{
    change: { value: string };
    readOnlyChangeAttempt: { value: string };
  }>();

  onMount(async () => {
    if (!element) while (!element) await new Promise((r) => setTimeout(r, 100));
    editor = new EditorView({
      extensions: [basicSetup, (html ? htmlLang : cssLang)(), oneDark],
      parent: element,
      doc: value,
    });
    value = editor.state.doc.toString();
    element.addEventListener('keypress', () => {
      value = editor.state.doc.toString();
    });
    setInterval(() => {
      const v = editor.state.doc.toString();
      if (v !== value) {
        if (!readonly) {
          dispatch('change', { value: v });
          value = v;
        } else {
          dispatch('readOnlyChangeAttempt', { value: v });
          editor.dispatch({
            changes: {
              from: 0,
              to: editor.state.doc.length,
              insert: value,
            },
          });
        }
      }
    }, 50);
  });
  $: {
    if (editor && value !== editor.state.doc.toString()) {
      editor.dispatch({
        changes: {
          from: 0,
          to: editor.state.doc.length,
          insert: value,
        },
      });
    }
  }
</script>

<div bind:this={element} class="codemirror-root" />

<style lang="scss">
  .codemirror-root {
    width: 100%;
    height: 100%;
    max-width: 100%;
    text-align: left;
    :global(.cm-editor) {
      border-radius: 4px;
    }
  }
</style>
