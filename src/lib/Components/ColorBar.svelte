<script>
  import anime from "animejs";
  import { onMount } from "svelte";

  const colors = ["#ffffff", "#4285f4", "#34A853", "#FBBC05", "#EA4335"];

  export let selectedColor = colors[0];

  export const unAnimateColors = () => {
    anime({
      targets: ".anime-color-circle",
      translateY: [0, -20],
      opacity: [1, 0],
      delay: anime.stagger(50),
      easing: "easeOutExpo",
      duration: 500,
    });
  };

  onMount(() => {
    anime({
      targets: ".anime-color-circle",
      translateY: [-50, 0],
      opacity: [0, 1],
      delay: anime.stagger(50),
      easing: "easeOutExpo",
      duration: 500,
    });
  });
</script>

{#each colors as color}
  <div
    class="h-full flex-1 flex items-center justify-center anime-color-circle"
  >
    <div
      class="rounded-full w-3 h-3 cursor-pointer"
      class:scale-150={color == selectedColor}
      class:opacity-80={color != selectedColor}
      style="background: {color}"
      on:click={() => (selectedColor = color)}
    />
  </div>
{/each}
