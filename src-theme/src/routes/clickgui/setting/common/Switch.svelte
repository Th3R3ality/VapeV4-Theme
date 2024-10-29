<script lang="ts">
    import {createEventDispatcher} from "svelte";

    export let value: boolean;
    export let name: string;

    const dispatch = createEventDispatcher();
</script>

<label class="switch-container">
    <div class="name">{name}</div>
    <div class="switch">
        <input type="checkbox" bind:checked={value} on:change={() => dispatch("change")}/>
        <span class="slider"></span>
    </div>
</label>

<style lang="scss">
  @use "sass:color";
  @import "../../../../colors.scss";

  .switch-container {
    display: flex;
    align-items: center;
    cursor: pointer;
  }

  .name {
    font-weight: 500;
    color: $clickgui-text-color;
    font-size: 12px;
    margin-left: 7px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .slider {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: #363536;
    transition: ease .2s;
    height: 12px;
    border-radius: 6px;
    width: 22px;

    &::before {
      position: absolute;
      content: "";
      height: 8px;
      width: 8px;
      top: 2px;
      left: 2px;
      background-color: $clickgui-base-color;
      border-radius: 50%;
      transition: ease .2s;
    }
  }

  .switch {
    position: relative;
    width: 22px;
    height: 12px;
    margin-left: auto;
    margin-right: 0;

    input {
      display: none;
    }

    input:checked + .slider {
      background-color: $accent-color
    }

    input:checked + .slider:before {
      transform: translateX(10px);
    }
  }
</style>
