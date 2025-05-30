<script lang="ts">
  import { Check, Funnel } from '@o7/icon/lucide';

  import { Badge } from '$lib/components/ui/badge';
  import { Button } from '$lib/components/ui/button';
  import * as Command from '$lib/components/ui/command';
  import * as Popover from '$lib/components/ui/popover';
  import { Separator } from '$lib/components/ui/separator';
  import { cn } from '$lib/utils';

  let {
    filterValues = $bindable<string[]>(),
    title,
    placeholder,
    disabled,
    options,
  }: {
    filterValues: string[];
    title: string;
    placeholder: string;
    disabled: boolean;
    options: { value: string; label: string }[];
  } = $props();

  let open = $state(false);

  function handleSelect(currentValue: string) {
    if (Array.isArray(filterValues) && filterValues.includes(currentValue)) {
      filterValues = filterValues.filter((v) => v !== currentValue);
    } else {
      filterValues = [
        ...(Array.isArray(filterValues) ? filterValues : []),
        currentValue,
      ];
    }
  }
</script>

<Popover.Root bind:open>
  <Popover.Trigger>
    {#snippet child({ props })}
      <Button
        variant="outline"
        size="sm"
        class="h-8 border-dashed"
        {...props}
        {disabled}
      >
        <Funnel size={16} class="mr-2" />
        {title}

        {#if filterValues.length > 0}
          <Separator orientation="vertical" class="mx-2 h-4" />
          <Badge
            variant="secondary"
            class="rounded-sm px-1 font-normal lg:hidden"
          >
            {filterValues.length}
          </Badge>
          <div class="hidden space-x-1 lg:flex">
            {#if filterValues.length > 2}
              <Badge variant="secondary" class="rounded-sm px-1 font-normal">
                {filterValues.length} Selected
              </Badge>
            {:else}
              {#each filterValues as option}
                <Badge variant="secondary" class="rounded-sm px-1 font-normal">
                  {option}
                </Badge>
              {/each}
            {/if}
          </div>
        {/if}
      </Button>
    {/snippet}
  </Popover.Trigger>
  <Popover.Content class="max-w-[400px] p-0" align="start" side="bottom">
    <Command.Root>
      <Command.Input {placeholder} />
      <Command.List>
        <Command.Viewport>
          <Command.Empty>No results found.</Command.Empty>
          <Command.Group>
            {#each options as option}
              <Command.Item
                value={option.value}
                keywords={[option.label]}
                onSelect={() => handleSelect(option.value)}
              >
                <div
                  class={cn(
                    'border-foreground flex size-5 items-center justify-center rounded-sm border',
                    filterValues.includes(option.value)
                      ? 'text-foreground'
                      : 'opacity-50 [&_svg]:invisible',
                  )}
                >
                  <Check className="size-4" />
                </div>
                <span class="truncate" title={option.label}>
                  {option.label}
                </span>
              </Command.Item>
            {/each}
          </Command.Group>
          {#if filterValues.length > 0}
            <Command.Separator />
            <Command.Item
              class="justify-center text-center"
              value="clear"
              onSelect={() => {
                filterValues = [];
              }}
            >
              Clear filters
            </Command.Item>
          {/if}
        </Command.Viewport>
      </Command.List>
    </Command.Root>
  </Popover.Content>
</Popover.Root>
