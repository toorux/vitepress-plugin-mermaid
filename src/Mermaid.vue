<template>
  <div class="v-mermaid-box" :class="[isFull ? 'is-full' : '']">
    <div v-html="svg" class="v-mermaid-container" :class="props.class"></div>
    <div class="full-btn" @click="isFull = !isFull" v-if="fullEnabled">
      <svg v-if="!isFull" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 448 512"><path fill="rgba(255, 255, 245, 0.86)" d="M32 32C14.3 32 0 46.3 0 64v96c0 17.7 14.3 32 32 32s32-14.3 32-32V96h64c17.7 0 32-14.3 32-32s-14.3-32-32-32H32zM64 352c0-17.7-14.3-32-32-32s-32 14.3-32 32v96c0 17.7 14.3 32 32 32h96c17.7 0 32-14.3 32-32s-14.3-32-32-32H64V352zM320 32c-17.7 0-32 14.3-32 32s14.3 32 32 32h64v64c0 17.7 14.3 32 32 32s32-14.3 32-32V64c0-17.7-14.3-32-32-32H320zM448 352c0-17.7-14.3-32-32-32s-32 14.3-32 32v64H320c-17.7 0-32 14.3-32 32s14.3 32 32 32h96c17.7 0 32-14.3 32-32V352z"/></svg>
      <svg v-else xmlns="http://www.w3.org/2000/svg" viewBox="0 0 448 512"><path fill="rgba(255, 255, 245, 0.86)"  d="M160 64c0-17.7-14.3-32-32-32s-32 14.3-32 32v64H32c-17.7 0-32 14.3-32 32s14.3 32 32 32h96c17.7 0 32-14.3 32-32V64zM32 320c-17.7 0-32 14.3-32 32s14.3 32 32 32H96v64c0 17.7 14.3 32 32 32s32-14.3 32-32V352c0-17.7-14.3-32-32-32H32zM352 64c0-17.7-14.3-32-32-32s-32 14.3-32 32v96c0 17.7 14.3 32 32 32h96c17.7 0 32-14.3 32-32s-14.3-32-32-32H352V64zM320 320c-17.7 0-32 14.3-32 32v96c0 17.7 14.3 32 32 32s32-14.3 32-32V384h64c17.7 0 32-14.3 32-32s-14.3-32-32-32H320z"/></svg>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref, toRaw, nextTick, watch } from "vue";
import { render, init } from "./mermaid";
import svgPanZoom from 'svg-pan-zoom';

//get mermaid settings
import { useData, defineClientComponent } from "vitepress";
//
// const svgPanZoom = defineClientComponent(() => {
//   return import('svg-pan-zoom');
// });


const pluginSettings = ref({
  securityLevel: "loose",
  startOnLoad: false,
  externalDiagrams: [],
});
const { page } = useData();
const { frontmatter } = toRaw(page.value);
const mermaidPageTheme = frontmatter.mermaidTheme || "";
const panZoomPageOption = frontmatter.panZoomOption || {};


const props = defineProps({
  graph: {
    type: String,
    required: true,
  },
  id: {
    type: String,
    required: true,
  },
  class:{
    type: String,
    required: false,
    default: "mermaid",
  },
  panZoomOption:{
    type: String,
    required: false,
    default: "{}",
  },
  maxHeight:{
    type: String,
    required: false,
    default: "20vh",
  }
});

const svg = ref(null);
const fullEnabled = ref(false);
const isFull = ref(false);
let mut = null;

onMounted(async () => {
  await init(pluginSettings.value.externalDiagrams);
  let settings = await import("virtual:mermaid-config");
  if (settings?.default) pluginSettings.value = settings.default;

  mut = new MutationObserver(async () => await renderChart());
  mut.observe(document.documentElement, { attributes: true });
  await renderChart();

  //refresh images on first render
  const hasImages =
    /<img([\w\W]+?)>/.exec(decodeURIComponent(props.graph))?.length > 0;
  if (hasImages)
    setTimeout(() => {
      let imgElements = document.getElementsByTagName("img");
      let imgs = Array.from(imgElements);
      if (imgs.length) {
        Promise.all(
          imgs
            .filter((img) => !img.complete)
            .map(
              (img) =>
                new Promise((resolve) => {
                  img.onload = img.onerror = resolve;
                })
            )
        ).then(async () => {
          await renderChart();
        });
      }
    }, 100);

  watch(isFull, async () => {
    await nextTick();
    await renderChart();
  });
});

onUnmounted(() => mut.disconnect());

const renderChart = async () => {
  const hasDarkClass = document.documentElement.classList.contains("dark");
  let mermaidConfig = {
    ...pluginSettings.value,
  };

  if (mermaidPageTheme) mermaidConfig.theme = mermaidPageTheme;
  if (hasDarkClass) mermaidConfig.theme = "dark";

  const graph = decodeURIComponent(props.graph)


  let svgCode = await render(
    props.id,
    graph,
    mermaidConfig
  );
  // This is a hack to force v-html to re-render, otherwise the diagram disappears
  // when **switching themes** or **reloading the page**.
  // The cause is that the diagram is deleted during rendering (out of Vue's knowledge).
  // Because svgCode does NOT change, v-html does not re-render.
  // This is not required for all diagrams, but it is required for c4c, mindmap and zenuml.
  const salt = Math.random().toString(36).substring(7);
  svg.value = `${svgCode} <span style="display: none">${salt}</span>`;


  await nextTick();
  const defaultPanZoomOption = {
    panEnabled: false,
    zoomEnabled: false,
    controlIconsEnabled: true,
  };
  const globalPanZoomOpt = JSON.parse(decodeURIComponent(props.panZoomOption));
  const graphPanZoomOpt = detectDirective(graph, 'panZoomOption').args ?? {};
  const panZoomOption = Object.assign(defaultPanZoomOption, globalPanZoomOpt, panZoomPageOption, graphPanZoomOpt);
  fullEnabled.value = panZoomOption.fullEnabled ?? false;
  if (!panZoomOption.panEnabled && !panZoomOption.zoomEnabled) return;
  const svgEle = document.getElementById(props.id);
  const minHeight = panZoomOption.minHeight || '20vh';
  const rectStyle = isFull.value ? `height: 100%;width: 100%;max-height: 100%;max-width: 100%;` : `min-height: ${minHeight}; max-width: 100%;`;
  svgEle.setAttribute('style', `${svgEle.getAttribute('style') ?? ''} ${rectStyle}`);
  const panZoomTiger = svgPanZoom(svgEle, panZoomOption);

  const resizeHandle = () => {
    panZoomTiger.resize();
    panZoomTiger.reset();
    panZoomTiger.updateBBox();
    // if (graph.startsWith('pie')) {
    //   panZoomTiger.fit();
    // }
    panZoomTiger.reset();
  }

  resizeHandle();

  window.addEventListener('resize', resizeHandle);

};

// refer https://github.com/mermaid-js/mermaid/blob/develop/packages/mermaid/src/utils.ts#L161
const detectDirective = function (
  text,
  type = null
) {
  const directiveWithoutOpen = /\s*(?:(\w+)(?=:):|(\w+))\s*(?:(\w+)|((?:(?!}%{2}).|\r?\n)*))?\s*(?:}%{2})?/gi;
  const directiveRegex = /%{2}{\s*(?:(\w+)\s*:|(\w+))\s*(?:(\w+)|((?:(?!}%{2}).|\r?\n)*))?\s*(?:}%{2})?/gi;
  try {
    const commentWithoutDirectives = new RegExp(
      `[%]{2}(?![{]${directiveWithoutOpen.source})(?=[}][%]{2}).*\n`,
      'ig'
    );
    text = text.trim().replace(commentWithoutDirectives, '').replace(/'/gm, '"');
    let match;
    const result = [];
    while ((match = directiveRegex.exec(text)) !== null) {
      // This is necessary to avoid infinite loops with zero-width matches
      if (match.index === directiveRegex.lastIndex) {
        directiveRegex.lastIndex++;
      }
      if (
        (match && !type) ||
        (type && match[1] && match[1].match(type)) ||
        (type && match[2] && match[2].match(type))
      ) {
        const type = match[1] ? match[1] : match[2];
        const args = match[3] ? match[3].trim() : match[4] ? JSON.parse(match[4].trim()) : null;
        result.push({ type, args });
      }
    }
    if (result.length === 0) {
      return { type: text, args: null };
    }

    return result.length === 1 ? result[0] : result;
  } catch (error) {
    console.error(
      `ERROR: ${
        (error).message
      } - Unable to parse directive type: '${type}' based on the text: '${text}'`
    );
    return { type: undefined, args: null };
  }
}


</script>

<style>

.v-mermaid-box {
  position: relative;
}

.v-mermaid-box.is-full {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(20px);
  z-index: 999;
}

.is-full > .v-mermaid-container {
  height: 100%;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.full-btn {
  position: absolute;
  width: 26px;
  height: 26px;
  top: 0;
  right: 0;
  overflow: hidden;
  padding: 4px 5px 5px;
  border-radius: 4px;
  background-color: rgba(139, 139, 140, 0.8);
  cursor: pointer;
}

.is-full .full-btn {
  top: 30px;
  right: 30px;
}

.full-btn:hover {
  background-color: rgba(52, 52, 52, 0.4);
}

.full-btn > svg {
  vertical-align: top;
}

</style>
