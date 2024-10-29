<script lang="ts">
    import {onMount} from "svelte";
    import {
        getModuleSettings,
        setModuleSettings,
        setModuleEnabled,
    } from "../../integration/rest";
    import type {ConfigurableSetting} from "../../integration/types";
    import GenericSetting from "./setting/common/GenericSetting.svelte";
    import {slide} from "svelte/transition";
    import {quintOut} from "svelte/easing";
    import {description as descriptionStore, highlightModuleName} from "./clickgui_store";
    import {setItem} from "../../integration/persistent_storage";
    import {convertToSpacedString, spaceSeperatedNames} from "../../theme/theme_config";
    import {scaleFactor} from "./clickgui_store";

    export let name: string;
    export let enabled: boolean;
    export let description: string;
    export let aliases: string[];

    let moduleNameElement: HTMLElement;
    let configurable: ConfigurableSetting;
    const path = `clickgui.${name}`;
    let expanded = false;

    onMount(async () => {
        configurable = await getModuleSettings(name);

        setTimeout(() => {
            expanded = localStorage.getItem(path) === "true"
        }, 500);
    });

    highlightModuleName.subscribe(() => {
        if (name !== $highlightModuleName) {
            return;
        }

        setTimeout(() => {
            if (!moduleNameElement) {
                return;
            }
            moduleNameElement.scrollIntoView({
                behavior: "smooth",
                block: "center",
            });
        }, 1000);
    });

    async function updateModuleSettings() {
        await setModuleSettings(name, configurable);
        configurable = await getModuleSettings(name);
    }

    async function toggleModule() {
        await setModuleEnabled(name, !enabled);
    }

    function setDescription() {
        const y = (moduleNameElement?.getBoundingClientRect().top ?? 0) + ((moduleNameElement?.clientHeight ?? 0) / 2);
        const x = moduleNameElement?.getBoundingClientRect().right ?? 0;
        let moduleDescription = description;
        if (aliases.length > 0) {
            moduleDescription += ` (aka ${aliases.map(a => $spaceSeperatedNames ? convertToSpacedString(a) : a).join(", ")})`;
        }
        descriptionStore.set({
            x: x * (2 / $scaleFactor),
            y: y * (2 / $scaleFactor),
            description: moduleDescription
        });
    }

    async function toggleExpanded() {
        expanded = !expanded;
        await setItem(path, expanded.toString());
    }
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
        class="module"
        class:expanded
        class:has-settings={configurable?.value.length > 2}
>
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <div
            class="name"
            on:contextmenu|preventDefault={toggleExpanded}
            on:click={toggleModule}
            on:mouseenter={setDescription}
            on:mouseleave={() => descriptionStore.set(null)}
            bind:this={moduleNameElement}
            class:enabled
            class:highlight={name === $highlightModuleName}
    >
        <span>
        {#if $spaceSeperatedNames}
            {convertToSpacedString(name)}
        {:else}
            {name}
        {/if}
        </span>
        <div class="expand-toggle"/>
    </div>


    {#if expanded && configurable}
        <div class="settings">
            {#each configurable.value as setting (setting.name)}
                <GenericSetting skipAnimationDelay={true} {path} bind:setting on:change={updateModuleSettings}/>
            {/each}
        </div>
    {/if}
</div>

<style lang="scss">
  @import "../../colors.scss";

  .module {
    position: relative;

    .name {
      cursor: pointer;

      color: $clickgui-text-dimmed-color;
      text-align: left;
      font-size: 14px;
      font-weight: 500;
      position: relative;
      padding: 11px;
      border-style: solid;
      border-bottom-color: $clickgui-base-color;
      border-width: 0 0 1px 0;

      &.highlight::before {
        content: "";
        position: absolute;
        top: 0;
        left: 0;
        width: calc(100% - 4px);
        height: calc(100% - 4px);
        border: solid 2px $accent-color;
      }

      &:hover {
        background-color: $clickgui-module-hover-color;
        color: $clickgui-text-color;
      }

      .expand-toggle {
        background-color: transparent;
        height: 30px;
        width: 30px;
        scale: 1 -1;
        position: absolute;
        background-image: url("/img/clickgui/icon-expand.png");
        background-repeat: no-repeat;
        overflow: clip;
        right: 5px;
        top: 5px;
        background-position: 50%;
      }

      &.enabled {
        color: $clickgui-text-enabled-color;
        background: $clickgui-module-enabled-color;
        border-style: solid;
        border-bottom-color: darken($clickgui-module-enabled-color, 10%);
        border-width: 0 0 1px 0;

        .expand-toggle{
          background-image: url("/img/clickgui/icon-expand-dark.png");
        }
      }
    }

    .settings {
      background-color: $clickgui-module-settings-background;
      padding: 0 11px 0 7px;
    }
  }


</style>
