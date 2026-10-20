<script lang="ts">
	import { page } from '$app/state';
	import type { Pipeline } from '$lib/modules/pipeline/types';
	import { methods as pipelineMethods } from '$lib/modules/pipeline/methods';
	import { pipelineStore } from '$lib/modules/pipeline/stores.svelte';
	import { audioStore } from '$lib/modules/audio/stores.svelte';
	import { ActivationButton, LatencyBadge, PipelineRateBadge, RunningTimer } from '$lib/modules/audio/ui';
	import Header from '$lib/components/layout/header.svelte';
	import Flow from '$lib/modules/flow';
	import { SnapshotHistory, SavedIndicator, UndoRedo } from '$lib/modules/flow/ui';
	import { Toaster } from 'svelte-french-toast';
	import { isFromFuture } from '$lib/modules/pipeline/migrations';
	import { Add } from '$lib/components/icons';

	let pipeline = $state<Pipeline | null>(null);
	let notFound = $state(false);
	let stale = $state(false);
	let busyMix = $state<string | null>(null);

	async function toggleMix(id: string, event: MouseEvent) {
		event.preventDefault();
		event.stopPropagation();
		if (busyMix) return;
		busyMix = id;
		try {
			if (audioStore.isRunning) {
				if (audioStore.runningPipelineId === id) await audioStore.deactivatePipeline();
			} else {
				const selected = await pipelineMethods.get(id);
				if (selected && !isFromFuture(selected)) await audioStore.activatePipeline(id, { nodes: selected.nodes, edges: selected.edges });
			}
		} catch (e) {
			audioStore.reportError(e);
		} finally {
			busyMix = null;
		}
	}

	$effect(() => {
		const id = page.params.id;
		if (!id) {
			notFound = true;
			return;
		}
		(async () => {
			const p = await pipelineMethods.get(id);
			if (!p) {
				notFound = true;
			} else if (isFromFuture(p)) {
				stale = true;
			} else {
				pipeline = p;
			}
		})();
	});

	const MAX_NAME_LENGTH = 64;
	const MIN_NAME_WIDTH_PX = 60;
	const NAME_EXTRA_PX = 4;

	let nameSaveTimer: ReturnType<typeof setTimeout> | undefined;
	function onNameInput() {
		if (pipeline && pipeline.name.length > MAX_NAME_LENGTH) {
			pipeline.name = pipeline.name.slice(0, MAX_NAME_LENGTH);
		}
		clearTimeout(nameSaveTimer);
		nameSaveTimer = setTimeout(() => {
			if (!pipeline) return;
			// Merge with the latest editor snapshot when one exists so we don't
			// clobber unsaved node/edge changes; otherwise persist the current
			// pipeline object with the new name.
			const snapshot = pipelineStore.editorActions?.getSnapshot();
			const next = snapshot ? { ...snapshot, name: pipeline.name } : { ...pipeline, updatedAt: Date.now() };
			pipelineStore.save(next);
		}, 500);
	}

	let nameInput = $state<HTMLInputElement | null>(null);
	let nameMirror = $state<HTMLSpanElement | null>(null);
	let nameWidth = $state(MIN_NAME_WIDTH_PX);

	$effect(() => {
		pipeline?.name;
		if (!nameMirror) return;
		let width = nameMirror.getBoundingClientRect().width;
		if (nameInput) {
			const cs = getComputedStyle(nameInput);
			width += parseFloat(cs.borderLeftWidth) + parseFloat(cs.borderRightWidth);
		}
		nameWidth = Math.max(MIN_NAME_WIDTH_PX, width + NAME_EXTRA_PX);
	});
</script>

<Toaster
	position="bottom-end"
	containerClassName="mr-72"
	toastOptions={{
		duration: 5000,
		className: 'bg-neutral-200! rounded-xl! text-neutral-900! px-3!'
	}} />

<Header>
	{#snippet left()}
		<div class="flex items-center gap-3">
			<a href="/" class="button-header px-4 text-sm">← Back to mixes</a>
			{#if pipeline}
				<div class="relative">
					<span bind:this={nameMirror} class="pointer-events-none invisible absolute top-0 left-0 px-2.5 whitespace-pre" aria-hidden="true">
						{pipeline.name}
					</span>
					<input
						bind:this={nameInput}
						bind:value={pipeline.name}
						oninput={onNameInput}
						maxlength={MAX_NAME_LENGTH}
						class="input-base !transition-none"
						style:width="{nameWidth}px" />
				</div>
			{/if}
		</div>
	{/snippet}

	{#snippet right()}
		<div class="flex items-center gap-3">
			{#if audioStore.isRunning}
				<RunningTimer />
				<LatencyBadge />
			{/if}
			<PipelineRateBadge />
			{#if pipeline}
				<UndoRedo />
				<SavedIndicator />
				<SnapshotHistory pipelineId={pipeline.id} />
				{#if pipeline.layout !== 'patchbay'}
					<ActivationButton pipelineId={pipeline.id} />
				{/if}
			{/if}
		</div>
	{/snippet}
</Header>

<div class="flex min-h-0 w-full flex-1">
	{#if notFound}
		<div class="p-8 text-sm text-gray-500">Pipeline not found.</div>
	{:else if stale}
		<div class="flex w-full flex-col items-start gap-4 p-8">
			<div class="warning-block max-w-xl">
				<span class="font-semibold">This pipeline was saved by a newer version of Splitwave.</span>
				<span>
					Its layout is not known to this build, so opening it would mean guessing at routing this version cannot represent. It is left untouched and
					can still be deleted.
				</span>
				<span>Update Splitwave to open it.</span>
			</div>
			<a href="/" class="button-main primary rounded-lg text-sm">Back to mixes</a>
		</div>
	{:else if pipeline}
		<aside class="flex w-56 shrink-0 flex-col border-r border-neutral-100 bg-background dark:border-[#11191a] dark:bg-[#1a2021]">
			<div class="border-b border-neutral-100 px-4 py-3 text-sm font-semibold dark:border-[#11191a]">Mixes</div>
			<div class="flex-1 overflow-y-auto p-2">
				{#each pipelineStore.pipelines as mix (mix.id)}
					{@const staleMix = isFromFuture(mix)}
					<div
						class={[
							'mb-1 flex items-center gap-1 rounded-lg px-2 py-2',
							mix.id === pipeline.id ? 'bg-neutral-300 dark:bg-[#303a3b]' : 'hover:bg-neutral-200 dark:hover:bg-[#252d2e]'
						]}>
						<a href={staleMix ? undefined : `/pipelines/${mix.id}`} class={['flex min-w-0 flex-1 items-center gap-2 text-sm', staleMix && 'opacity-50']}>
							<span class={['size-2 shrink-0 rounded-full', audioStore.runningPipelineId === mix.id ? 'bg-emerald-400' : 'bg-neutral-500']}></span>
							<span class="truncate">{mix.name}</span>
						</a>
						<button
							class={[
								'h-6 min-w-9 rounded border px-1.5 text-[10px] font-semibold',
								audioStore.runningPipelineId === mix.id
									? 'border-emerald-500/50 bg-emerald-500/15 text-emerald-700 dark:text-emerald-300'
									: 'border-neutral-400 bg-neutral-100 text-neutral-800 dark:bg-[#202728] dark:text-neutral-200'
							]}
							disabled={staleMix || !!busyMix || (audioStore.isRunning && audioStore.runningPipelineId !== mix.id)}
							aria-label={`${audioStore.runningPipelineId === mix.id ? 'Stop' : 'Start'} ${mix.name}`}
							onclick={(event) => toggleMix(mix.id, event)}>
							{busyMix === mix.id ? '…' : audioStore.runningPipelineId === mix.id ? 'On' : 'Off'}
						</button>
					</div>
				{/each}
			</div>
			<div class="flex flex-col gap-2 border-t border-neutral-100 p-3 dark:border-[#11191a]">
				<a href="/" class="button-main primary justify-start gap-2 rounded-lg px-2.5 py-1.5 text-xs"><Add class="size-3.5" /> New mix</a>
				<a href="/virtual-devices" class="px-2 text-xs text-neutral-900 hover:text-theme">Manage virtual devices</a>
			</div>
		</aside>
		{#key pipeline.id}
			<Flow.ui.Flow {pipeline} />
		{/key}
	{:else}
		<div class="p-8 text-sm text-gray-500">Loading…</div>
	{/if}
</div>
