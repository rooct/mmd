<script lang="ts">
  import Editor from '$lib/components/Editor.svelte';
  // import Navbar from '$lib/components/Navbar.svelte';
  // import Preset from '$lib/components/Preset.svelte';
  // import Actions from '$lib/components/Actions.svelte';
  // import View from '$lib/components/View.svelte';
  import Card from '$lib/components/Card/Card.svelte';
  // import History from '$lib/components/History/History.svelte';
  import { stateStore, updateCodeStore } from '$lib/util/state';
  import { initHandler } from '$lib/util/util';
  import { onMount } from 'svelte';
  import { base } from '$app/paths';
  import { dev } from '$app/environment';
  import type { Tab, DocumentationConfig, EditorMode, ValidatedState } from '$lib/types';
  import { deserializeState } from '$lib/util/serde';
  import Theme from '$lib/components/Theme.svelte';
  import View from '$lib/components/View.svelte';

  const MCBaseURL = dev ? 'http://localhost:5174' : 'https://mermaidchart.com';
  const docURLBase = 'https://mermaid-js.github.io/mermaid';
  const docMap: DocumentationConfig = {
    graph: {
      code: '/#/flowchart',
      config: '/#/flowchart?id=configuration'
    },
    flowchart: {
      code: '/#/flowchart',
      config: '/#/flowchart?id=configuration'
    },
    sequenceDiagram: {
      code: '/#/sequenceDiagram',
      config: '/#/sequenceDiagram?id=configuration'
    },
    classDiagram: {
      code: '/#/classDiagram',
      config: '/#/classDiagram?id=configuration'
    },
    'stateDiagram-v2': {
      code: '/#/stateDiagram'
    },
    gantt: {
      code: '/#/gantt',
      config: '/#/gantt?id=configuration'
    },
    pie: {
      code: '/#/pie'
    },
    erDiagram: {
      code: '/#/entityRelationshipDiagram',
      config: '/#/entityRelationshipDiagram?id=styling'
    },
    journey: {
      code: '/#/user-journey'
    },
    gitGraph: {
      code: '/#/gitgraph',
      config: '/#/gitgraph?id=gitgraph-specific-configuration-options'
    }
  };
  let docURL = docURLBase;
  let activeTabID = 'code';
  let docKey = '';
  let isView = false;
  stateStore.subscribe(({ code, editorMode }: ValidatedState) => {
    activeTabID = editorMode;
    const codeTypeMatch = /(\S+)\s/.exec(code);
    if (codeTypeMatch && codeTypeMatch.length > 1) {
      docKey = codeTypeMatch[1];
      const docConfig = docMap[docKey] ?? { code: '' };
      docURL = docURLBase + (docConfig[editorMode] ?? docConfig.code ?? '');
    }
  });

  const tabSelectHandler = (message: CustomEvent<Tab>) => {
    const editorMode: EditorMode = message.detail.id === 'code' ? 'code' : 'config';
    updateCodeStore({ editorMode });
  };

  const tabs: Tab[] = [
    {
      id: 'code',
      title: 'Code',
      icon: 'fas fa-code'
    },
    {
      id: 'config',
      title: 'Config',
      icon: 'fas fa-cogs'
    }
  ];

  onMount(async () => {
    await initHandler();
    const resizer = document.querySelector<HTMLElement>('#resizeHandler');
    const element = document.querySelector<HTMLElement>('#editorPane');
    if (!resizer || !element) {
      console.debug('Failed to find resize handler or editor pane', { resizer, element });
      return;
    }
    const resize = ({ pageX }: { pageX: number }) => {
      const newWidth = pageX - element.getBoundingClientRect().left;
      if (newWidth > 50) {
        element.style.width = `${newWidth}px`;
      }
    };

    const stopResize = () => {
      window.removeEventListener('mousemove', resize);
    };
    resizer.addEventListener('mousedown', (event) => {
      event.preventDefault();
      window.addEventListener('mousemove', resize);
      window.addEventListener('mouseup', stopResize);
    });
  });
  const setView = () => {
    isView = !isView;
  };
</script>

<div class="flex h-full flex-col overflow-hidden">
  <div class="flex flex-1 overflow-hidden">
    <div class="flex flex-col" id="editorPane" style="width: {isView ? '50%' : '100%'}">
      <Card on:select={tabSelectHandler} {tabs} isCloseable={false} {activeTabID} title="">
        <div slot="actions" class="flex items-center">
          <!-- <div class="form-control flex-row items-center">
            <label class="label cursor-pointer" for="autoSync">
              <span>sync</span>
              <input
                type="checkbox"
                class="toggle {$stateStore.autoSync ? 'btn-secondary' : 'toggle-primary'} ml-1"
                id="autoSync"
                bind:checked={$inputStateStore.autoSync} />
            </label>
          </div> -->

          <!-- {#if !$stateStore.autoSync}
            <button
              class="btn btn-secondary btn-xs mr-1"
              title="Sync Diagram ({cmdKey} + Enter)"
              data-cy="sync"
              on:click={syncDiagram}><i class="fas fa-sync" /></button>
          {/if} -->
          <Theme />
          <div class="  hidden items-center gap-2 md:flex">
            <button
              class="btn btn-secondary btn-xs"
              title="View documentation for {docKey.replace('Diagram', '')} diagram">
              <a target="_blank" href="https://mermaid.js.org/intro/" data-cy="docs">
                <i class="fas fa-book mr-1" />Docs
              </a>
            </button>
            <button
              class="btn btn-secondary btn-xs"
              title="View documentation for {docKey.replace('Diagram', '')} diagram">
              <a
                target="_blank"
                rel="noreferrer"
                class="flex-grow"
                href={`https://mermaid.ink/img/${$stateStore.serialized}?type=png`}>
                PNG
              </a>
            </button>
            <button
              class="btn btn-secondary btn-xs"
              title="View documentation for {docKey.replace('Diagram', '')} diagram">
              <a
                target="_blank"
                rel="noreferrer"
                href={`https://mermaid.ink/svg/${$stateStore.serialized}`}>
                SVG
              </a>
            </button>
            <a
              href={`${base}/show#${$stateStore.serialized}`}
              target="_blank"
              class="gap btn btn-secondary btn-xs"
              title="View diagram in new page"><i class="fas fa-share" />预览</a>

            <button class="btn btn-secondary btn-xs" on:click={setView}
              ><i class="fas fa-stop-circle"></i></button>
          </div>

          <div class=" dropdown">
            <button
              class="btn btn-ghost block md:hidden"
              title="View documentation for {docKey.replace('Diagram', '')} diagram">
              <i class="fas fa-ellipsis-h" />
            </button>
            <div
              class=" dropdown-content right-0 top-px mt-12 flex w-32 flex-col gap-2 overflow-y-auto bg-white p-3 text-base-content shadow-2xl">
              <button
                class="btn btn-secondary btn-xs"
                title="View documentation for {docKey.replace('Diagram', '')} diagram">
                <a target="_blank" href="https://mermaid.js.org/intro/" data-cy="docs">
                  <i class="fas fa-book mr-1" />Docs
                </a>
              </button>
              <button
                class="btn btn-secondary btn-xs"
                title="View documentation for {docKey.replace('Diagram', '')} diagram">
                <a
                  target="_blank"
                  rel="noreferrer"
                  class="flex-grow"
                  href={`https://mermaid.ink/img/${$stateStore.serialized}?type=png`}>
                  PNG
                </a>
              </button>
              <button
                class="btn btn-secondary btn-xs"
                title="View documentation for {docKey.replace('Diagram', '')} diagram">
                <a
                  target="_blank"
                  rel="noreferrer"
                  href={`https://mermaid.ink/svg/${$stateStore.serialized}`}>
                  SVG
                </a>
              </button>
              <a
                href={`${base}/show#${$stateStore.serialized}`}
                target="_blank"
                class="gap btn btn-secondary btn-xs"
                title="View diagram in new page"><i class="fas fa-share" />预览</a>

              <button class="btn btn-secondary btn-xs" on:click={setView}
                ><i class="fas fa-stop-circle"></i></button>
            </div>
          </div>
        </div>

        <Editor />
      </Card>
    </div>
    <div id="resizeHandler" class="hidden md:block" />
    {#if isView}
      <div class="flex flex-1 flex-col overflow-hidden">
        <Card title="Diagram" isCloseable={false}>
          <div class="flex-1 overflow-auto">
            <View />
          </div>
        </Card>
      </div>
    {/if}
  </div>
</div>

<style>
  #resizeHandler {
    cursor: col-resize;
    padding: 0 2px;
  }

  #resizeHandler::after {
    width: 2px;
    height: 100%;
    top: 0;
    content: '';
    position: absolute;
    background-color: hsla(var(--b3));
    margin-left: -1px;
    transition-duration: 0.2s;
  }

  #resizeHandler:hover::after {
    margin-left: -2px;
    background-color: hsla(var(--p));
    width: 4px;
  }
</style>
