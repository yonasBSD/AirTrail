<script lang="ts">
  import { LoaderCircle, Upload } from '@o7/icon/lucide';
  import { toast } from 'svelte-sonner';

  import { PageHeader } from '../';

  import PlatformTabs from './PlatformTabs.svelte';

  import { platforms } from './';

  import { Button } from '$lib/components/ui/button';
  import { Card } from '$lib/components/ui/card';
  import { Checkbox } from '$lib/components/ui/checkbox';
  import { Label } from '$lib/components/ui/label';
  import { ScrollArea } from '$lib/components/ui/scroll-area';
  import { processFile } from '$lib/import';
  import { trpc } from '$lib/trpc';
  import { cn } from '$lib/utils';

  let { open = $bindable() }: { open: boolean } = $props();

  let files: FileList | null = $state(null);
  let fileError: string | null = $state(null);

  const validateFile = () => {
    const file = files?.[0];
    if (!file) return;

    if (
      !file.name.endsWith('.csv') &&
      !file.name.endsWith('.txt') &&
      !file.name.endsWith('.json')
    ) {
      fileError = 'File type not supported';
    } else if (file.size > 5 * 1024 * 1024) {
      fileError = 'File must be less than 5MB';
    } else {
      fileError = null;
    }
  };

  const canImport = $derived(!!files?.[0] && !fileError);
  const invalidator = {
    onSuccess: () => {
      trpc.flight.list.utils.invalidate();
    },
  };
  const createMany = trpc.flight.createMany.mutation(invalidator);

  let importing = $state(false);
  let unknownAirports = $state<string[]>([]);
  const handleImport = async () => {
    const file = files?.[0];
    if (!file || fileError) return;

    importing = true;
    let result: Awaited<ReturnType<typeof processFile>>;
    try {
      result = await processFile(file, platform.value, {
        filterOwner: ownerOnly,
        airlineFromFlightNumber: matchAirlineFromFlightNumber,
      });
    } catch (error) {
      toast.error('Failed to import file');
      console.error(error);
      importing = false;
      return;
    }

    const { flights } = result;

    if (!flights.length) {
      toast.error('No flights found in the file');
      files = null;
      importing = false;
      return;
    }

    await $createMany.mutateAsync(flights);

    toast.success(`Imported ${flights.length} flights`);
    unknownAirports = result.unknownAirports;
    files = null;
    importing = false;
    if (!unknownAirports.length) {
      open = false;
    }
  };

  let platform = $state<(typeof platforms)[0]>(platforms[0]);
  let ownerOnly = $state(false);
  let matchAirlineFromFlightNumber = $state(true);
</script>

<PageHeader title="Import">
  {#snippet subtitleHtml()}
    <p class="text-muted-foreground text-sm">
      Import your data from another platform. Learn more about the platforms <a
        href="https://airtrail.johan.ohly.dk/docs/features/import"
        target="_blank"
        class="text-blue-500 underline">in the documentation</a
      >.
    </p>
  {/snippet}
  {#if !unknownAirports.length}
    <PlatformTabs bind:platform />
    {#if platform.options.airlineFromFlightNumber}
      <div class="flex items-center gap-2">
        <Checkbox
          id="match-airline-from-flight-number"
          bind:checked={matchAirlineFromFlightNumber}
          aria-labelledby="match-airline-from-flight-number-label"
        />
        <Label
          id="match-airline-from-flight-number-label"
          for="match-airline-from-flight-number"
        >
          Match airline from flight number
        </Label>
      </div>
    {/if}
    {#if platform.options.filterOwner}
      <div class="flex items-center gap-2">
        <Checkbox
          id="owner-only"
          bind:checked={ownerOnly}
          aria-labelledby="owner-only-label"
        />
        <Label id="owner-only-label" for="owner-only">
          Only import your flights
        </Label>
      </div>
    {/if}
    <label for="file" class="block">
      <Card
        class={cn(
          'cursor-pointer py-12 border-2 border-dashed flex flex-col items-center hover:bg-card-hover dark:hover:bg-dark-2',
          { 'border-destructive': fileError },
        )}
      >
        <Upload />
        {#if fileError}
          {fileError}
        {:else}
          {files?.[0]?.name ?? 'Upload file'}
        {/if}
      </Card>
    </label>
    <input
      onchange={validateFile}
      id="file"
      name="file"
      type="file"
      accept=".csv,.txt,.json"
      bind:files
      class="hidden"
    />
    <Button onclick={handleImport} disabled={!canImport || importing}>
      {#if importing}
        <LoaderCircle class="animate-spin mr-1" size={16} />
      {/if}
      Import
    </Button>
  {:else}
    <h3 class="text-lg font-semibold">Unknown airports</h3>
    <p class="text-muted-foreground">
      The following airports are not in the database and flights with these
      airports have therefore not been imported.
    </p>
    <p class="text-muted-foreground">
      Chances are the airport codes have been officially changed, but that the
      change hasn't reflected in your export file. The easiest solution is to
      investigate the codes, and manually change the occurrences in the file
      before trying to import again.
    </p>
    <p class="text-muted-foreground">
      If the airports are truly missing, please report them directly to our
      source, <a href="https://ourairports.com/">OurAirports</a>, or add them as
      custom airports.
    </p>
    <ScrollArea class="h-[30dvh]">
      <ul class="mt-4 ml-4 list-disc">
        {#each unknownAirports as airport}
          <li>{airport}</li>
        {/each}
      </ul>
    </ScrollArea>
    <Button href="https://ourairports.com/" target="_blank">OurAirports</Button>
    <Button onclick={() => (unknownAirports = [])} variant="secondary">
      Close
    </Button>
  {/if}
</PageHeader>
