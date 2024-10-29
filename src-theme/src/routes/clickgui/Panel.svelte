<script lang="ts">
    import {onMount} from "svelte";
    import type {Module as TModule} from "../../integration/types";
    import {listen} from "../../integration/ws";
    import Module from "./Module.svelte";
    import type {ToggleModuleEvent} from "../../integration/events";
    import {fly} from "svelte/transition";
    import {quintOut} from "svelte/easing";
    import {highlightModuleName, maxPanelZIndex} from "./clickgui_store";
    import {setItem} from "../../integration/persistent_storage";
    import {scaleFactor} from "./clickgui_store";

    export let category: string;
    export let modules: TModule[];
    export let panelIndex: number;

    let panelElement: HTMLElement;
    let modulesElement: HTMLElement;

    let renderedModules: TModule[] = [];

    let moving = false;
    let prevX = 0;
    let prevY = 0;
    const panelConfig = loadPanelConfig();

    interface PanelConfig {
        top: number;
        left: number;
        expanded: boolean;
        scrollTop: number;
        zIndex: number;
    }

    function clamp(number: number, min: number, max: number) {
        return Math.max(min, Math.min(number, max));
    }

    function loadPanelConfig(): PanelConfig {
        const localStorageItem = localStorage.getItem(
            `clickgui.panel.${category}`,
        );

        if (!localStorageItem) {
            return {
                top: panelIndex * 50 + 20,
                left: 20,
                expanded: false,
                scrollTop: 0,
                zIndex: 0
            };
        } else {
            const config: PanelConfig = JSON.parse(localStorageItem);

            // Migration
            if (!config.zIndex) {
                config.zIndex = 0;
            }

            if (config.zIndex > $maxPanelZIndex) {
                $maxPanelZIndex = config.zIndex;
            }

            if (config.expanded) {
                renderedModules = modules;
            }

            return config;
        }
    }

    async function savePanelConfig() {
        await setItem(
            `clickgui.panel.${category}`,
            JSON.stringify(panelConfig),
        );
    }

    function fixPosition() {
        panelConfig.left = clamp(panelConfig.left, 0, document.documentElement.clientWidth * (2 / $scaleFactor) - panelElement.offsetWidth);
        panelConfig.top = clamp(panelConfig.top, 0, document.documentElement.clientHeight * (2 / $scaleFactor) - panelElement.offsetHeight);
    }

    function onMouseDown() {
        moving = true;

        panelConfig.zIndex = ++$maxPanelZIndex;
    }

    function onMouseMove(e: MouseEvent) {
        if (moving) {
            panelConfig.left += (e.screenX - prevX) * (2 / $scaleFactor);
            panelConfig.top += (e.screenY - prevY) * (2 / $scaleFactor);
        }

        prevX = e.screenX;
        prevY = e.screenY;

        fixPosition();
        savePanelConfig();
    }

    function onMouseUp() {
        moving = false;
    }

    function toggleExpanded() {
        if (panelConfig.expanded) {
            renderedModules = [];
        } else {
            renderedModules = modules;
        }

        panelConfig.expanded = !panelConfig.expanded;

        setTimeout(() => {
            fixPosition();
            savePanelConfig();
        }, 500);
    }

    function handleModulesScroll() {
        panelConfig.scrollTop = modulesElement.scrollTop;
        savePanelConfig();
    }

    highlightModuleName.subscribe(() => {
        const highlightModule = modules.find(
            (m) => m.name === $highlightModuleName,
        );
        if (highlightModule) {
            panelConfig.zIndex = ++$maxPanelZIndex;
            panelConfig.expanded = true;
            renderedModules = modules;
            savePanelConfig();
        }
    });

    listen("toggleModule", (e: ToggleModuleEvent) => {
        const moduleName = e.moduleName;
        const moduleEnabled = e.enabled;

        const mod = modules.find((m) => m.name === moduleName);
        if (!mod) return;

        mod.enabled = moduleEnabled;
        modules = modules;
        if (panelConfig.expanded) {
            renderedModules = modules;
        }
    });

    onMount(() => {
        setTimeout(() => {
            if (!modulesElement) {
                return;
            }

            modulesElement.scrollTo({
                top: panelConfig.scrollTop,
                behavior: "instant"
            })
        }, 100);
    });
</script>

<svelte:window on:mouseup={onMouseUp} on:mousemove={onMouseMove}/>

<div
        class="panel"
        style="left: {panelConfig.left}px; top: {panelConfig.top}px; z-index: {panelConfig.zIndex};"
        bind:this={panelElement}
>
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div
            class="title"
            on:mousedown={onMouseDown}
            on:contextmenu|preventDefault={toggleExpanded}
    >
        <object
            data="img/clickgui/icon-{category.toLowerCase()}.svg"
            type="image/svg+xml" height="15">
        <img
                src="img/clickgui/icon-{category.toLowerCase()}.png"
                alt="icon"
                height="15"
        />
        </object>
        <span class="category">{category}</span>

        <button class="expand-toggle" on:click={toggleExpanded}>
            <div class="icon" class:expanded={panelConfig.expanded}></div>
        </button>
    </div>

    <div class="modules" on:scroll={handleModulesScroll} bind:this={modulesElement}>
        {#each renderedModules as {name, enabled, description, aliases} (name)}
            <Module {name} {enabled} {description} {aliases}/>
        {/each}
    </div>
</div>

<style lang="scss">
  @import "../../colors.scss";

  .panel {
    border-radius: 5px;
    width: 220px;
    position: absolute;
    overflow: hidden;
    box-shadow: inset 0 0 5px #161616, 0 0 10px #000a;
    will-change: transform;
    border-style: solid;
    border-width: 1.1px;
    border-color: #2d2d2d;
    background: #1A191A;
  }

  .title {
    display: grid;
    grid-template-columns: max-content 1fr max-content;
    align-items: center;
    column-gap: 12px;
    padding: 11px 11px;
    cursor: grab;

    .category {
      font-size: 14px;
      color: $clickgui-text-color;
      font-weight: 500;
    }
  }

  .modules {
    max-height: 545px;
    overflow-y: auto;
    overflow-x: hidden;
    background-color: rgba($clickgui-base-color, 0.8);
  }

  .modules::-webkit-scrollbar {
    width: 0;
  }

  .expand-toggle {
    background-color: transparent;
    border: none;
    cursor: pointer;

    .icon {
      height: 16px;
      width: 12px;
      scale: 1 -1;
      position: relative;
      background-image: url("/img/clickgui/icon-panel-arrow.png");

      &.expanded {
        transform: scaleY(-1);
      }
    }

    .icon:hover{
      background-image: url("/img/clickgui/icon-panel-arrow-hover.png");
    }
  }
</style>
