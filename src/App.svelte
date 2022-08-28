<script>
  import anime from "animejs";
  import { onMount } from "svelte";
  import { appDir, BaseDirectory } from "@tauri-apps/api/path";
  import ColorBar from "./lib/Components/ColorBar.svelte";
  import { clickOutside } from "./lib/Events/click_outside";
  import Flatpickr from "svelte-flatpickr";
  import "flatpickr/dist/flatpickr.css";
  import "flatpickr/dist/themes/dark.css";
  import Clock from "./lib/Icons/clock.icon.svelte";
  import { nanoid } from "nanoid";
  import dayjs from "dayjs";

  import { fly } from "svelte/transition";

  import relativeTime from "dayjs/plugin/relativeTime";
  import { createDir, readTextFile, writeTextFile } from "@tauri-apps/api/fs";
  dayjs.extend(relativeTime);

  let tasks = [];
  let value;
  let datePickerOpen = false;
  let inputFocused;
  let unAnimateColors;
  let date;
  let color;

  let taskInitialDelay = 700;
  let taskStaggerOffset = 100;

  onMount(async () => {
    anime({
      targets: ".animate",
      translateY: [100, 0],
      opacity: [0, 1],
      delay: anime.stagger(50),
      easing: "easeInOutExpo",
      duration: 1000,
      complete: () => {
        taskInitialDelay = 0;
        taskStaggerOffset = 0;
      },
    });

    createDir("data", { dir: BaseDirectory.App, recursive: true }).then(
      async () => {
        readTextFile("data.json", { dir: BaseDirectory.App }).then((data) => {
          tasks = JSON.parse(data);
        });
      }
    );
  });

  const focusInput = () => {
    if (inputFocused) return;

    anime({
      targets: ".color-bar",
      height: [0, 30],
      easing: "easeInOutExpo",
      complete: () => (inputFocused = true),
      duration: 500,
    });
  };

  const unfocusInput = () => {
    if (!datePickerOpen && unAnimateColors && inputFocused) {
      unAnimateColors();
      anime({
        targets: ".color-bar",
        height: [30, 0],
        easing: "easeInOutExpo",
        complete: () => (inputFocused = false),
      });
    }
  };

  const flatpickrOptions = {
    onChange: (selectedDates, dateStr, instance) => {
      console.log("test");
      date = selectedDates[0];
    },
    onOpen: () => (datePickerOpen = true),
    onClose: () => (datePickerOpen = false),
  };

  const handleKeypress = (event) => {
    if (event.key === "Enter") {
      event.preventDefault();

      createTask();
    }
  };

  const createTask = async () => {
    if (!value) return;

    tasks = [
      ...tasks,
      {
        id: nanoid(),
        title: value,
        status: "todo",
        color: color,
        deadline: date,
      },
    ];

    await saveToDisk();
    value = "";
  };

  const moveTask = async (id, nextStatus) => {
    tasks[tasks.findIndex((t) => t.id == id)]["status"] = nextStatus;
    await saveToDisk();
  };

  const saveToDisk = async () => {
    try {
      await writeTextFile("data.json", JSON.stringify(tasks), {
        dir: BaseDirectory.App,
      });
    } catch (e) {
      console.log(e);
    }
  };

  const deleteTask = (id) => {
    tasks = tasks.filter((t) => t.id != id);
  };
</script>

<div
  class="px-5 pt-5  flex justify-between items-center"
  data-tauri-drag-region
>
  <div class="text-3xl pointer-events-none">Todos</div>
  <div class="flex gap-2">
    <div class="w-3 h-3 rounded-full bg-[#FBBC05]" />
    <div class="w-3 h-3 rounded-full bg-[#EA4335]" />
  </div>
</div>

<main class="p-5">
  <div use:clickOutside on:click_outside={unfocusInput} on:click={focusInput}>
    <div class="flex animate items-center gap-2">
      <input
        bind:value
        class="flex-grow px-2 py-1  outline-none bg-zinc-800 text-zinc-300"
        on:keypress={handleKeypress}
      />

      <div class="w-8 h-8  bg-zinc-800 relative pointer-events-none">
        <Flatpickr
          options={flatpickrOptions}
          class="w-full h-full  opacity-0 pointer-events-auto"
        />

        <div
          class="w-full h-full absolute top-0 left-0 flex items-center justify-center opacity-50"
        >
          <Clock />
        </div>
      </div>

      <div
        class="w-8 h-8  bg-zinc-800 flex items-center justify-center"
        on:click={createTask}
      >
        <span class="opacity-50">✓</span>
      </div>
    </div>

    <!-- Color picker -->
    <div class="h-full w-full flex mt-3  color-bar">
      {#if inputFocused}
        <ColorBar bind:selectedColor={color} bind:unAnimateColors />
      {/if}
    </div>
  </div>

  <div>
    <div class="mt-8 text-lg font-bold mb-1 animate">todo</div>

    {#each tasks.filter((t) => t.status === "todo") as task, i}
      <div
        class="w-full flex items-center gap-3 mt-2"
        in:fly={{ y: 20, delay: taskInitialDelay + taskStaggerOffset * i }}
      >
        <div
          class="w-3 h-3 flex-shrink-0 cursor-pointer"
          on:click={() => moveTask(task.id, "inprogress")}
          style="background-color: {task.color};"
        />

        <div class="w-full group flex flex-col ">
          <div
            class=" w-full text-sm opacity-80 leading-tight flex justify-between items-center"
          >
            <div>
              {task.title}
            </div>

            <div
              class="opacity-0 group-hover:opacity-50 cursor-pointer text-lg leading-none tracking-tighter relative"
              on:click={() => deleteTask(task.id)}
            >
              ×
            </div>
          </div>

          <div class="text-xs opacity-50">
            {#if task.deadline}
              Deadline {dayjs(task.deadline).fromNow()}
            {:else}
              No deadline
            {/if}
          </div>
        </div>
      </div>
    {/each}
  </div>

  <div>
    <div class="mt-10 font-bold mb-1 animate">in progress</div>

    {#each tasks.filter((t) => t.status === "inprogress") as task, i}
      <div
        class="w-full flex items-center gap-2 mt-2"
        in:fly={{ y: 50, delay: taskInitialDelay + taskStaggerOffset * i }}
      >
        <div
          class="w-3 h-3 flex-shrink-0 animate-pulse rounded-full cursor-pointer"
          style="background-color: {task.color};"
          on:click={() => moveTask(task.id, "done")}
        />

        <div class="group w-full flex flex-col ">
          <div
            class=" w-full text-sm opacity-80 leading-tight flex justify-between items-center"
          >
            <div>
              {task.title}
            </div>

            <div
              class="opacity-0 group-hover:opacity-50 cursor-pointer text-lg leading-none tracking-tighter relative"
              on:click={() => deleteTask(task.id)}
            >
              ×
            </div>
          </div>

          <div class="text-xs opacity-50">
            {#if task.deadline}
              Deadline {dayjs(task.deadline).fromNow()}
            {:else}
              No deadline
            {/if}
          </div>
        </div>
      </div>
    {/each}
  </div>

  <div class="mb-10">
    <div class="mt-10 font-bold mb-1 animate">
      done <span class="font-thin opacity-40">
        ({tasks.filter((t) => t.status === "done").length})
      </span>
    </div>

    {#each tasks.filter((t) => t.status === "done") as task, i}
      <div
        class="w-full flex items-center gap-2 mt-2"
        in:fly={{ y: 50, delay: taskInitialDelay + taskStaggerOffset * i }}
      >
        <div
          class="w-3 h-3 flex-shrink-0 leading-none "
          style="color: {task.color};"
        >
          ▲
        </div>

        <div class="w-full flex justify-between group">
          <div class="text-sm opacity-80 leading-tight">{task.title}</div>

          <div
            class="opacity-0 group-hover:opacity-50 cursor-pointer text-lg leading-none tracking-tighter relative"
            on:click={() => deleteTask(task.id)}
          >
            ×
          </div>
        </div>
      </div>
    {/each}
  </div>
</main>
