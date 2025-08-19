<script lang="ts">
  import favicon from "$lib/assets/favicon.svg";

  let { children } = $props();
</script>

<svelte:head>
  <link rel="icon" href={favicon} />
  <link
    rel="stylesheet"
    href="https://fixedsys.javanile.org/css/fixedsys-dark.css"
  />
</svelte:head>

{@render children?.()}
