<script lang="ts">
  import autoAnimate from '@formkit/auto-animate';
  import { createCombobox, melt } from '@melt-ui/svelte';
  import { CircleX, ChevronsUpDown } from '@o7/icon/lucide';
  import { writable } from 'svelte/store';
  import { fly } from 'svelte/transition';
  import type { SuperForm } from 'sveltekit-superforms';
  import { z } from 'zod';

  import * as Form from '$lib/components/ui/form';
  import { AIRLINES } from '$lib/data/airlines';
  import { sortAndFilterByMatch } from '$lib/utils';
  import { type Airline, airlineFromICAO } from '$lib/utils/data/airlines';
  import type { flightSchema } from '$lib/zod/flight';

  let {
    form,
  }: {
    form: SuperForm<z.infer<typeof flightSchema>>;
  } = $props();
  const { form: formData } = form;

  const selected = writable(
    $formData.airline
      ? {
          label: airlineFromICAO($formData.airline)?.name,
          value: $formData.airline,
        }
      : undefined,
  );

  const {
    elements: { menu, input, option },
    states: { open, inputValue },
  } = createCombobox<string>({
    forceVisible: true,
    selected,
    onSelectedChange: ({ next }) => {
      if (next?.value) $formData.airline = next.value;
      return next;
    },
  });

  // If the field is updated externally, update the selected value
  formData.subscribe(() => {
    if ($formData.airline === $selected?.value) return;
    selected.set(
      $formData.airline
        ? {
            label: airlineFromICAO($formData.airline)?.name,
            value: $formData.airline,
          }
        : undefined,
    );
  });

  $effect(() => {
    if (!$open) {
      $inputValue = $selected?.label ?? '';
    }
  });

  let airlines: Airline[] = $state([]);
  $effect(() => {
    if ($open && $inputValue !== '') {
      airlines = sortAndFilterByMatch(AIRLINES, $inputValue, [
        { key: 'icao', exact: true },
        { key: 'iata', exact: true },
        { key: 'name', exact: false },
      ]);
    } else {
      airlines = [];
    }
  });
</script>

<Form.Field {form} name="airline" class="flex flex-col">
  <Form.Control>
    {#snippet children({ props })}
      <Form.Label>Airline</Form.Label>
      <div class="relative">
        <input
          use:melt={$input}
          class="border-input bg-background ring-offset-background placeholder:text-muted-foreground focus-visible:ring-ring flex h-10 w-full rounded-md border px-3 py-2 text-sm focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2 disabled:opacity-50 pr-12"
          placeholder="Select airline"
        />
        {#if $open && $selected}
          <button
            transition:fly={{ duration: 200, x: 20 }}
            type="button"
            onclick={() => {
              // @ts-expect-error - This is totally fine
              $selected = undefined;
              $inputValue = '';
            }}
            class="cursor-pointer absolute right-10 top-1/2 z-10 -translate-y-1/2 text-muted-foreground hover:text-foreground"
          >
            <CircleX size="20" />
          </button>
        {/if}
        <div
          class="absolute right-2 top-1/2 z-10 -translate-y-1/2 text-muted-foreground"
        >
          <ChevronsUpDown class="size-4" />
        </div>
      </div>
      <input hidden bind:value={$formData.airline} name={props.name} />
    {/snippet}
  </Form.Control>
  {#if $open}
    <ul
      class="z-[5000] flex max-h-[300px] flex-col overflow-hidden rounded-lg border"
      use:melt={$menu}
      transition:fly={{ duration: 150, y: -5 }}
    >
      <!-- svelte-ignore a11y-no-noninteractive-tabindex -->
      <div
        use:autoAnimate
        class="flex max-h-full flex-col gap-1 overflow-y-auto bg-popover text-card-foreground"
        tabindex="0"
      >
        {#each airlines as airline}
          <li
            use:melt={$option({
              value: airline.icao,
              label: airline.name,
            })}
            class="relative cursor-pointer scroll-my-2 rounded-md p-2 dark:bg-dark-1 border data-[highlighted]:bg-zinc-300 data-[highlighted]:dark:bg-dark-2"
          >
            <div class="flex flex-col overflow-hidden">
              <span class="truncate">{airline.name}</span>
              <p class="text-sm">
                {#if airline.iata}
                  <span class="text-muted-foreground">IATA</span>
                  <b class="mr-2">{airline.iata}</b>
                {/if}
                <span class="text-muted-foreground">ICAO</span>
                <b>{airline.icao}</b>
              </p>
            </div>
          </li>
        {:else}
          <li
            class="relative cursor-pointer scroll-my-2 rounded-md p-2
        bg-popover dark:bg-dark-1 border"
          >
            {#if $inputValue}
              No airlines found.
            {:else}
              Start typing to search...
            {/if}
          </li>
        {/each}
      </div>
    </ul>
  {/if}
  <Form.FieldErrors />
</Form.Field>
