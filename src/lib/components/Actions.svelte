<script lang="ts">
  import { browser } from '$app/environment';
  import Card from '$lib/components/Card/Card.svelte';
  import { env } from '$lib/util/env';
  import { pakoSerde } from '$lib/util/serde';
  import { stateStore } from '$lib/util/state';
  import { logEvent } from '$lib/util/stats';
  import { toBase64 } from 'js-base64';
  import dayjs from 'dayjs';

  const { krokiRendererUrl, rendererUrl } = env;
  type Exporter = (context: CanvasRenderingContext2D, image: HTMLImageElement) => () => void;

  const getFileName = (extension: string) =>
    `mermaid-diagram-${dayjs().format('YYYY-MM-DD-HHmmss')}.${extension}`;

  const getBase64SVG = (svg?: HTMLElement, width?: number, height?: number): string => {
    if (svg) {
      // Prevents the SVG size of the interface from being changed
      svg = svg.cloneNode(true) as HTMLElement;
    }
    height && svg?.setAttribute('height', `${height}px`);
    width && svg?.setAttribute('width', `${width}px`); // Workaround https://stackoverflow.com/questions/28690643/firefox-error-rendering-an-svg-image-to-html5-canvas-with-drawimage
    if (!svg) {
      svg = getSvgElement();
    }
    const svgString = svg.outerHTML
      .replaceAll('<br>', '<br/>')
      .replaceAll(/<img([^>]*)>/g, (m, g: string) => `<img ${g} />`);
    return toBase64(svgString);
  };

  const exportImage = (event: Event, exporter: Exporter) => {
    const canvas: HTMLCanvasElement = document.createElement('canvas');
    const svg = document.querySelector<HTMLElement>('#container svg');
    if (!svg) {
      throw new Error('svg not found');
    }
    const box: DOMRect = svg.getBoundingClientRect();
    canvas.width = box.width;
    canvas.height = box.height;
    if (imagemodeselected === 'width') {
      const ratio = box.height / box.width;
      canvas.width = userimagesize;
      canvas.height = userimagesize * ratio;
    } else if (imagemodeselected === 'height') {
      const ratio = box.width / box.height;
      canvas.width = userimagesize * ratio;
      canvas.height = userimagesize;
    }

    const context = canvas.getContext('2d');
    if (!context) {
      throw new Error('context not found');
    }
    context.fillStyle = `hsl(${window.getComputedStyle(document.body).getPropertyValue('--b1')})`;
    context.fillRect(0, 0, canvas.width, canvas.height);

    const image = new Image();
    image.addEventListener('load', exporter(context, image));
    image.src = `data:image/svg+xml;base64,${getBase64SVG(svg, canvas.width, canvas.height)}`;

    event.stopPropagation();
    event.preventDefault();
  };

  const getSvgElement = () => {
    const svgElement = document.querySelector('#container svg')?.cloneNode(true) as HTMLElement;
    svgElement.setAttribute('xmlns:xlink', 'http://www.w3.org/1999/xlink');
    const fontAwesomeCdnUrl = [...document.head.querySelectorAll('link')]
      .map((link) => link.href)
      .find((url) => url.includes('font-awesome'));
    if (fontAwesomeCdnUrl == null) {
      return svgElement;
    }
    const styleElement = document.createElement('style');
    styleElement.textContent = `@import url("${fontAwesomeCdnUrl}");'`;
    svgElement.prepend(styleElement);
    return svgElement;
  };

  const simulateDownload = (download: string, href: string): void => {
    const a = document.createElement('a');
    a.download = download;
    a.href = href;
    a.click();
    a.remove();
  };
  const downloadImage: Exporter = (context, image) => {
    return () => {
      const { canvas } = context;
      context.drawImage(image, 0, 0, canvas.width, canvas.height);
      simulateDownload(
        getFileName('png'),
        canvas.toDataURL('image/png').replace('image/png', 'image/octet-stream')
      );
    };
  };

  const isClipboardAvailable = (): boolean => {
    return Object.prototype.hasOwnProperty.call(window, 'ClipboardItem') as boolean;
  };

  const clipboardCopy: Exporter = (context, image) => {
    return () => {
      const { canvas } = context;
      context.drawImage(image, 0, 0, canvas.width, canvas.height);
      canvas.toBlob((blob) => {
        try {
          if (!blob) {
            throw new Error('blob is empty');
          }
          void navigator.clipboard.write([
            new ClipboardItem({
              [blob.type]: blob
            })
          ]);
        } catch (error) {
          console.error(error);
        }
      });
    };
  };

  const onCopyClipboard = (event: Event) => {
    exportImage(event, clipboardCopy);
    logEvent('copyClipboard');
  };

  const onDownloadPNG = (event: Event) => {
    exportImage(event, downloadImage);
    logEvent('download', {
      type: 'png'
    });
  };

  const onDownloadSVG = () => {
    simulateDownload(getFileName('svg'), `data:image/svg+xml;base64,${getBase64SVG()}`);
    logEvent('download', {
      type: 'svg'
    });
  };

  const onCopyMarkdown = () => {
    document.querySelector<HTMLInputElement>('#markdown')?.select();
    document.execCommand('Copy');
    logEvent('copyMarkdown');
  };

  let gistURL = '';
  stateStore.subscribe(({ loader }) => {
    if (loader?.type === 'gist') {
      // @ts-expect-error Gist will have url
      gistURL = loader.config.url;
    }
  });

  const loadGist = () => {
    if (!gistURL) {
      alert('Please enter a Gist URL first');
    }
    window.location.href = `${window.location.pathname}?gist=${gistURL}`;
    logEvent('loadGist');
  };

  let iUrl: string;
  let svgUrl: string;
  let krokiUrl: string;
  let mdCode: string;
  let imagemodeselected = 'auto';
  let userimagesize = 1080;

  let isNetlify = false;
  if (browser && ['mermaid.live', 'netlify'].some((path) => window.location.host.includes(path))) {
    isNetlify = true;
  }
  stateStore.subscribe(({ code, serialized }) => {
    iUrl = `https://mermaid.ink/img/${serialized}?type=png`;
    svgUrl = `https://mermaid.ink/svg/${serialized}`;
    krokiUrl = `${krokiRendererUrl}/mermaid/svg/${pakoSerde.serialize(code)}`;
    mdCode = `[![](${iUrl})](${window.location.protocol}//${window.location.host}${window.location.pathname}#${serialized})`;
  });
</script>

<Card title="Actions" isOpen={false}>
  <div class="m-1 flex flex-wrap gap-1">
    <button id="downloadPNG" class="action-btn flex-grow" on:click={onDownloadPNG}>
      <i class="fas fa-download mr-1" /> PNG
    </button>
    <button id="downloadSVG" class="action-btn flex-grow" on:click={onDownloadSVG}>
      <i class="fas fa-download mr-2" /> SVG
    </button>
    <a target="_blank" rel="noreferrer" class="flex-grow" href={iUrl}>
      <button class="action-btn w-full">
        <i class="fas fa-external-link-alt mr-2" /> PNG
      </button>
    </a>
    <a target="_blank" rel="noreferrer" class="flex-grow" href={svgUrl}>
      <button class="action-btn w-full">
        <i class="fas fa-external-link-alt mr-2" /> SVG
      </button>
    </a>
  </div></Card>
