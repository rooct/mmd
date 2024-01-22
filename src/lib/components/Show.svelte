<script lang="ts">
  import { inputStateStore, stateStore, updateCodeStore } from '$lib/util/state';
  import { onMount } from 'svelte';
  import panzoom from 'svg-pan-zoom';
  import type { State } from '$lib/types';
  import { logEvent } from '$lib/util/stats';
  import { cmdKey } from '$lib/util/util';
  import { render as renderDiagram } from '$lib/util/mermaid';
  import type { MermaidConfig } from 'mermaid';

  let config = '';
  let container: HTMLDivElement;
  let view: HTMLDivElement;
  let error = false;
  let errorLines: string[] = [];
  let outOfSync = false;
  let hide = false;
  let manualUpdate = true;
  let panZoomEnabled = true;
  let pzoom: typeof panzoom | undefined;

  const handlePanZoomChange = () => {
    if (!pzoom) {
      return;
    }
    const pan = pzoom.getPan();
    const zoom = pzoom.getZoom();
    updateCodeStore({ pan, zoom });
    logEvent('panZoom');
  };
  let codeValue = localStorage.getItem('codeStore') || '';
  const updateCodeValue = () => {
    codeValue = localStorage.getItem('codeStore') || '';
    void handleStateChange(JSON.parse(codeValue));
  };

  const handlePanZoom = (state: State) => {
    if (!state.panZoom) {
      return;
    }
    hide = true;
    pzoom?.destroy();
    pzoom = undefined;
    void Promise.resolve().then(() => {
      const graphDiv = document.querySelector<HTMLElement>('#graph-div');
      if (!graphDiv) {
        return;
      }
      pzoom = panzoom(graphDiv, {
        onPan: handlePanZoomChange,
        onZoom: handlePanZoomChange,
        controlIconsEnabled: true,
        fit: true,
        center: true
      });
      hide = false;
    });
  };

  const handleStateChange = async (state) => {
    if (state.error !== undefined) {
      error = true;
      errorLines = state.error.toString().split('\n');
      return;
    }
    error = false;
    try {
      if (container && state && (state.updateDiagram || state.autoSync)) {
        if (!state.autoSync) {
          $inputStateStore.updateDiagram = false;
        }
        outOfSync = false;
        manualUpdate = true;
        // Do not render if there is no change in Code/Config/PanZoom
        if (config === JSON.parse(state.mermaid) && panZoomEnabled === state.panZoom) {
          return;
        }
        config = JSON.parse(state.mermaid);
        const scroll = view.parentElement?.scrollTop;

        delete container.dataset.processed;

        const { svg, bindFunctions } = await renderDiagram(
          Object.assign({}, JSON.parse(state.mermaid)) as MermaidConfig,
          state.code,
          'graph-div'
        );

        if (svg.length > 0) {
          handlePanZoom(state);
          container.innerHTML = svg;
          const graphDiv = document.querySelector<HTMLElement>('#graph-div');
          if (!graphDiv) {
            throw new Error('graph-div not found');
          }

          graphDiv.setAttribute('height', '100%');
          graphDiv.style.maxWidth = '100%';

          if (bindFunctions) {
            bindFunctions(graphDiv);
          }
        }
        if (view.parentElement && scroll) {
          view.parentElement.scrollTop = scroll;
        }
        error = false;
      } else if (manualUpdate) {
        manualUpdate = false;
      } else if (config !== state.mermaid) {
        outOfSync = true;
      }
    } catch (error_) {
      console.error('view fail', error_);
      error = true;
    }
  };

  onMount(() => {
    void handleStateChange(JSON.parse(codeValue));
    window.addEventListener('storage', updateCodeValue);
  });
</script>

{#if (error && $stateStore.error instanceof Error) || outOfSync}
  <div
    class="absolute z-10 w-full p-2 font-mono {error
      ? 'text-red-600'
      : 'text-yellow-600'} bg-base-100 bg-opacity-80 text-left"
    id="errorContainer">
    {#if error}
      {#each errorLines as line}
        {line}<br />
      {/each}
    {:else}
      Diagram out of sync. <br />
      Press <i class="fas fa-sync" /> (Sync button) or <kbd>{cmdKey} + Enter</kbd> to sync.
    {/if}
  </div>
{/if}

<div id="view" bind:this={view} class="h-full p-2" class:error class:outOfSync>
  <div id="container" bind:this={container} class="h-full overflow-auto" class:hide />
</div>

<style>
  #view {
    flex: 1;
  }

  #container {
    transition: visibility 0.3s;
  }

  .error,
  .outOfSync {
    opacity: 0.5;
  }

  .hide {
    visibility: hidden;
  }
</style>
