<script lang="ts">
    import {fade} from "svelte/transition";
</script>

<img class="logo" src="img/wurst-logo.png" alt="logo" out:fade|global={{duration: 250}}>

<style lang="scss">
  .logo {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
  }
</style>
