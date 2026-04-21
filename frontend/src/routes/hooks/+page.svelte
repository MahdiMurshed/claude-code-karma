<script lang="ts">
	import {
		Webhook,
		FolderOpen,
		ShieldAlert,
		ChevronsUpDown,
		ChevronsDownUp,
		Puzzle
	} from 'lucide-svelte';
	import PageHeader from '$lib/components/layout/PageHeader.svelte';
	import StatsGrid from '$lib/components/StatsGrid.svelte';
	import SegmentedControl from '$lib/components/ui/SegmentedControl.svelte';
	import CollapsibleGroup from '$lib/components/ui/CollapsibleGroup.svelte';
	import EmptyState from '$lib/components/ui/EmptyState.svelte';
	import HookEventNode from '$lib/components/hooks/HookEventNode.svelte';
	import HookScriptCard from '$lib/components/hooks/HookScriptCard.svelte';
	import type { StatItem, HookEventSummary } from '$lib/api-types';
	import { getHookSourceColorVars } from '$lib/utils';

	let { data } = $props();

	// View mode state
	let viewMode = $state<'timeline' | 'sources'>('timeline');

	// Stats for hero section
	let stats = $derived<StatItem[]>([
		{
			title: 'Sources',
			value: data.hooks.stats.total_sources.toLocaleString(),
			icon: FolderOpen,
			color: 'orange'
		},
		{
			title: 'Registrations',
			value: data.hooks.stats.total_registrations.toLocaleString(),
			icon: Webhook,
			color: 'blue'
		},
		{
			title: 'Can Block',
			value: data.hooks.stats.blocking_hooks.toLocaleString(),
			icon: ShieldAlert,
			color: 'orange'
		}
	]);

	// Phase ordering for timeline view
	const PHASE_ORDER = [
		'Session Lifecycle',
		'User Input',
		'Tool Lifecycle',
		'Agent Lifecycle',
		'Context & Permissions',
		'Session End',
		'Setup'
	];

	// Phase accent inks — each phase takes a muted editorial ink,
	// reused for the 6px dot marker next to the mono-caps eyebrow.
	const PHASE_INK: Record<string, string> = {
		'Session Lifecycle': 'var(--nav-orange)',
		'User Input': 'var(--nav-blue)',
		'Tool Lifecycle': 'var(--nav-green)',
		'Agent Lifecycle': 'var(--nav-purple)',
		'Context & Permissions': 'var(--nav-teal)',
		'Session End': 'var(--nav-gray)',
		Setup: 'var(--text-muted)'
	};

	function phaseInk(phase: string): string {
		return PHASE_INK[phase] ?? 'var(--text-muted)';
	}

	// Group events by phase
	interface PhaseGroup {
		phase: string;
		events: HookEventSummary[];
	}

	let eventsByPhase = $derived.by<PhaseGroup[]>(() => {
		const grouped = new Map<string, HookEventSummary[]>();

		for (const event of data.hooks.event_summaries) {
			const phase = event.phase || 'Other';
			if (!grouped.has(phase)) {
				grouped.set(phase, []);
			}
			grouped.get(phase)!.push(event);
		}

		// Sort by phase order
		const result: PhaseGroup[] = [];
		for (const phase of PHASE_ORDER) {
			if (grouped.has(phase)) {
				result.push({ phase, events: grouped.get(phase)! });
				grouped.delete(phase);
			}
		}

		// Add any remaining phases not in the order
		for (const [phase, events] of grouped.entries()) {
			result.push({ phase, events });
		}

		return result;
	});

	// Track expanded state for timeline events
	let expandedEvents = $state<Set<string>>(new Set());

	function toggleEvent(eventType: string) {
		if (expandedEvents.has(eventType)) {
			expandedEvents.delete(eventType);
		} else {
			expandedEvents.add(eventType);
		}
		expandedEvents = new Set(expandedEvents);
	}

	let allEventsExpanded = $derived(
		data.hooks.event_summaries.length > 0 &&
			data.hooks.event_summaries.every((e) => expandedEvents.has(e.event_type))
	);

	function expandAllEvents() {
		expandedEvents = new Set(data.hooks.event_summaries.map((e) => e.event_type));
	}

	function collapseAllEvents() {
		expandedEvents = new Set();
	}

	function toggleAllEvents() {
		if (allEventsExpanded) {
			collapseAllEvents();
		} else {
			expandAllEvents();
		}
	}

	// Track expanded state for source groups
	let expandedSources = $state<Set<string>>(new Set());

	function toggleSource(sourceId: string) {
		if (expandedSources.has(sourceId)) {
			expandedSources.delete(sourceId);
		} else {
			expandedSources.add(sourceId);
		}
		expandedSources = new Set(expandedSources);
	}

	let hasHooks = $derived(data.hooks.sources.length > 0 || data.hooks.event_summaries.length > 0);
</script>

<div class="space-y-8">
	<!-- Page Header -->
	<PageHeader
		title="Hooks"
		icon={Webhook}
		iconColor="--nav-orange"
		breadcrumbs={[{ label: 'Dashboard', href: '/' }, { label: 'Hooks' }]}
		subtitle="Hook scripts intercepting your Claude Code sessions"
	/>

	<!-- Hero Stats -->
	{#if hasHooks}
		<div
			class="relative overflow-hidden rounded-[var(--radius-md)] p-8 border border-[var(--border)]"
			style="background: linear-gradient(135deg, var(--bg-base) 0%, var(--nav-orange-subtle) 100%);"
		>
			<div class="relative">
				<StatsGrid {stats} columns={3} />
			</div>
		</div>
	{/if}

	<!-- View Switcher -->
	{#if hasHooks}
		<div class="flex items-center justify-between">
			<SegmentedControl
				options={[
					{ label: 'Event Timeline', value: 'timeline' },
					{ label: 'By Source', value: 'sources' }
				]}
				bind:value={viewMode}
			/>

			{#if viewMode === 'timeline' && data.hooks.event_summaries.length > 0}
				<button
					onclick={toggleAllEvents}
					class="expand-all-btn"
					title={allEventsExpanded ? 'Collapse all events' : 'Expand all events'}
				>
					{#if allEventsExpanded}
						<ChevronsDownUp size={14} strokeWidth={1.75} />
						<span>Collapse All</span>
					{:else}
						<ChevronsUpDown size={14} strokeWidth={1.75} />
						<span>Expand All</span>
					{/if}
				</button>
			{/if}
		</div>
	{/if}

	<!-- Content Area -->
	{#if !hasHooks}
		<EmptyState
			icon={Webhook}
			title="No hooks found"
			description="Hook scripts will appear here once you configure them in ~/.claude/settings.json or through a plugin."
		/>
	{:else if viewMode === 'timeline'}
		<!-- Timeline View -->
		<div class="space-y-10">
			{#each eventsByPhase as phaseGroup, i (phaseGroup.phase)}
				<section>
					<!-- Hairline rule above every phase except the first -->
					{#if i > 0}
						<div class="rule mb-8"></div>
					{/if}

					<!-- Phase Eyebrow + dot -->
					<h2 class="phase-eyebrow">
						<span
							class="phase-eyebrow__dot"
							style="background: {phaseInk(phaseGroup.phase)};"
							aria-hidden="true"
						></span>
						<span>{phaseGroup.phase}</span>
						<span class="phase-eyebrow__count">
							{phaseGroup.events.length}
						</span>
					</h2>

					<!-- Events in this phase -->
					<div class="space-y-0">
						{#each phaseGroup.events as event (event.event_type)}
							<HookEventNode
								{event}
								open={expandedEvents.has(event.event_type)}
								onToggle={() => toggleEvent(event.event_type)}
							/>
						{/each}
					</div>
				</section>
			{/each}
		</div>
	{:else}
		<!-- By Source View -->
		<div class="space-y-4">
			{#each data.hooks.sources as source (source.source_id)}
				{@const sourceColors = getHookSourceColorVars(
					source.source_type,
					source.source_name
				)}
				<CollapsibleGroup
					title={source.source_name}
					open={expandedSources.has(source.source_id)}
					onOpenChange={() => toggleSource(source.source_id)}
					accentColor={sourceColors.color}
				>
					{#snippet icon()}
						<div
							class="source-icon"
							style="background-color: {sourceColors.subtle}; color: {sourceColors.color};"
						>
							{#if source.source_type === 'plugin'}
								<Puzzle size={14} strokeWidth={1.75} />
							{:else}
								<FolderOpen size={14} strokeWidth={1.75} />
							{/if}
						</div>
					{/snippet}
					{#snippet metadata()}
						<div class="flex items-center gap-3">
							<span class="source-meta tabular-nums">
								{source.total_registrations} registration{source.total_registrations !==
								1
									? 's'
									: ''}
							</span>
							<span class="source-meta-sep" aria-hidden="true">·</span>
							<span class="source-meta tabular-nums">
								{source.event_types_covered.length} event type{source
									.event_types_covered.length !== 1
									? 's'
									: ''}
							</span>
							{#if source.blocking_hooks_count > 0}
								<span class="source-block-pill tabular-nums">
									{source.blocking_hooks_count} blocking
								</span>
							{/if}
						</div>
					{/snippet}

					<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
						{#each source.scripts as script (script.filename)}
							<HookScriptCard
								{script}
								sourceType={source.source_type}
								sourceName={source.source_name}
							/>
						{/each}
					</div>
				</CollapsibleGroup>
			{/each}
		</div>
	{/if}
</div>

<style>
	.expand-all-btn {
		display: inline-flex;
		align-items: center;
		gap: 8px;
		padding: 6px 12px;
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.16em;
		text-transform: uppercase;
		font-weight: 500;
		color: var(--text-secondary);
		background: var(--bg-base);
		border: 1px solid var(--border);
		border-radius: var(--radius-sm);
		cursor: pointer;
		transition:
			color var(--duration-fast) var(--ease),
			border-color var(--duration-fast) var(--ease),
			background var(--duration-fast) var(--ease);
	}

	.expand-all-btn:hover {
		color: var(--text-primary);
		border-color: var(--border-hover);
		background: var(--bg-subtle);
	}

	.phase-eyebrow {
		display: flex;
		align-items: center;
		gap: 10px;
		margin: 0 0 18px;
		font-family: var(--font-mono);
		font-size: 11px;
		font-weight: 500;
		letter-spacing: 0.18em;
		text-transform: uppercase;
		color: var(--text-secondary);
	}

	.phase-eyebrow__dot {
		width: 6px;
		height: 6px;
		border-radius: 50%;
		flex-shrink: 0;
	}

	.phase-eyebrow__count {
		margin-left: 2px;
		padding: 0 6px;
		font-size: 10px;
		letter-spacing: 0.1em;
		color: var(--text-muted);
		background: var(--bg-muted);
		border-radius: 999px;
		font-variant-numeric: tabular-nums;
		line-height: 1.7;
	}

	.source-icon {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		width: 32px;
		height: 32px;
		border-radius: var(--radius-sm);
	}

	.source-meta {
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.12em;
		text-transform: uppercase;
		color: var(--text-muted);
	}

	.source-meta-sep {
		color: var(--text-faint);
		font-size: 10px;
	}

	.source-block-pill {
		display: inline-flex;
		align-items: center;
		padding: 2px 7px;
		font-family: var(--font-mono);
		font-size: 10px;
		font-weight: 600;
		letter-spacing: 0.14em;
		text-transform: uppercase;
		background: var(--error-subtle);
		color: var(--error);
		border-radius: var(--radius-xs);
	}

	.rule {
		height: 1px;
		background: var(--border);
	}
</style>
