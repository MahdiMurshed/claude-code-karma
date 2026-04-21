<script lang="ts">
	import { Puzzle, Search, Bot, Zap, Terminal } from 'lucide-svelte';
	import PageHeader from '$lib/components/layout/PageHeader.svelte';
	import StatsGrid from '$lib/components/StatsGrid.svelte';
	import PluginCard from '$lib/components/plugins/PluginCard.svelte';
	import { SkeletonPluginCard } from '$lib/components/skeleton';
	import { API_BASE } from '$lib/config';
	import type { PluginSummary, PluginsOverview, StatItem } from '$lib/api-types';

	let { data } = $props();

	// Check if data is still loading
	let isLoading = $derived(!data.plugins && !data.error);

	// Progressive usage loading state
	let usageLoaded = $state(false);
	let pluginUsageMap = $state<Record<string, { total_runs: number; estimated_cost_usd: number }>>(
		{}
	);

	// Fetch full usage stats in the background after initial render
	$effect(() => {
		if (data.plugins && typeof window !== 'undefined') {
			fetch(`${API_BASE}/plugins?include_usage=true`)
				.then((r) => r.json())
				.then((full: PluginsOverview) => {
					const map: Record<string, { total_runs: number; estimated_cost_usd: number }> =
						{};
					for (const p of full.plugins) {
						map[p.name] = {
							total_runs: p.total_runs,
							estimated_cost_usd: p.estimated_cost_usd
						};
					}
					pluginUsageMap = map;
					usageLoaded = true;
				})
				.catch(() => {
					usageLoaded = true;
				});
		}
	});

	// Merge usage data into the initial plugin list once loaded
	let enrichedPlugins = $derived.by<PluginSummary[]>(() => {
		const plugins = data.plugins?.plugins || [];
		if (!usageLoaded) return plugins;
		return plugins.map((p) => ({
			...p,
			total_runs: pluginUsageMap[p.name]?.total_runs ?? p.total_runs,
			estimated_cost_usd: pluginUsageMap[p.name]?.estimated_cost_usd ?? p.estimated_cost_usd
		}));
	});

	// Filter state
	let searchQuery = $state('');
	let selectedFilter = $state<'all' | 'official' | 'community'>('all');

	type FilterKey = 'all' | 'official' | 'community';
	const filterOptions: { label: string; value: FilterKey }[] = [
		{ label: 'All', value: 'all' },
		{ label: 'Official', value: 'official' },
		{ label: 'Community', value: 'community' }
	];

	// Compute stats for hero section — brass accent for plugin count,
	// StatsCard palette (blue/green/teal) for the breakdown. No 'violet' key
	// exists in StatColor; using 'accent' gives brass ink on the masthead row.
	let stats = $derived.by<StatItem[]>(() => {
		const plugins = enrichedPlugins;
		const totalPlugins = plugins.length;
		const totalAgents = plugins.reduce((sum, p) => sum + p.agent_count, 0);
		const totalSkills = plugins.reduce((sum, p) => sum + p.skill_count, 0);
		const totalCommands = plugins.reduce((sum, p) => sum + (p.command_count || 0), 0);

		return [
			{ title: 'Plugins', value: totalPlugins, icon: Puzzle, color: 'accent' },
			{ title: 'Agents', value: totalAgents, icon: Bot, color: 'blue' },
			{ title: 'Skills', value: totalSkills, icon: Zap, color: 'green' },
			{ title: 'Commands', value: totalCommands, icon: Terminal, color: 'teal' }
		];
	});

	// Filter plugins
	let filteredPlugins = $derived.by<PluginSummary[]>(() => {
		let plugins = enrichedPlugins;

		// Filter by type
		if (selectedFilter === 'official') {
			plugins = plugins.filter((p) => p.is_official);
		} else if (selectedFilter === 'community') {
			plugins = plugins.filter((p) => !p.is_official);
		}

		// Filter by search
		if (searchQuery.trim()) {
			const query = searchQuery.toLowerCase();
			plugins = plugins.filter(
				(p) =>
					p.name.toLowerCase().includes(query) ||
					(p.description && p.description.toLowerCase().includes(query))
			);
		}

		return plugins;
	});

	let hasPlugins = $derived(enrichedPlugins.length > 0);
	let hasFilteredPlugins = $derived(filteredPlugins.length > 0);

	let officialCount = $derived(enrichedPlugins.filter((p) => p.is_official).length);
	let communityCount = $derived(enrichedPlugins.filter((p) => !p.is_official).length);

	function filterCount(key: FilterKey): number {
		if (key === 'all') return enrichedPlugins.length;
		if (key === 'official') return officialCount;
		return communityCount;
	}
</script>

<div class="space-y-8">
	<!-- Page Header -->
	<PageHeader
		title="Plugins"
		icon={Puzzle}
		iconColor="--nav-violet"
		breadcrumbs={[{ label: 'Dashboard', href: '/' }, { label: 'Plugins' }]}
		subtitle="View and manage Claude Code plugins"
	/>

	<!-- Hero Stats — flat paper, hairline rule, no gradient -->
	{#if hasPlugins}
		<section class="plugins-hero">
			<div class="plugins-hero__eyebrow">
				<span class="dot"></span>
				<span>Installed · overview</span>
			</div>
			<StatsGrid {stats} columns={4} />
		</section>
	{/if}

	<!-- Filters Row -->
	<div class="plugins-filters">
		<div class="plugins-filter-group" role="radiogroup" aria-label="Filter plugins">
			{#each filterOptions as option}
				{@const isActive = selectedFilter === option.value}
				<button
					type="button"
					role="radio"
					aria-checked={isActive}
					class="plugins-filter-chip"
					class:is-active={isActive}
					onclick={() => (selectedFilter = option.value)}
				>
					<span>{option.label}</span>
					<span class="plugins-filter-count">{filterCount(option.value)}</span>
				</button>
			{/each}
		</div>

		<div class="plugins-search">
			<Search
				class="plugins-search__icon"
				size={14}
				strokeWidth={1.75}
			/>
			<input
				type="text"
				bind:value={searchQuery}
				placeholder="Search plugins..."
				class="plugins-search__input"
			/>
		</div>
	</div>

	<!-- Content -->
	{#if isLoading}
		<!-- Loading Skeleton -->
		<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-5">
			{#each Array(6) as _}
				<SkeletonPluginCard />
			{/each}
		</div>
	{:else if data.error}
		<div class="plugins-empty" role="alert">
			<Puzzle size={36} strokeWidth={1.5} class="plugins-empty__icon" />
			<p class="plugins-empty__title">Error loading plugins</p>
			<p class="plugins-empty__desc">{data.error}</p>
		</div>
	{:else if !hasPlugins}
		<div class="plugins-empty">
			<Puzzle size={36} strokeWidth={1.5} class="plugins-empty__icon" />
			<p class="plugins-empty__title">No plugins installed</p>
			<p class="plugins-empty__desc">Install plugins via Claude Code to see them here.</p>
		</div>
	{:else if !hasFilteredPlugins}
		<div class="plugins-empty">
			<Search size={36} strokeWidth={1.5} class="plugins-empty__icon" />
			<p class="plugins-empty__title">No matching plugins</p>
			<p class="plugins-empty__desc">Try adjusting your search or filter.</p>
		</div>
	{:else}
		<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-5 stagger-children">
			{#each filteredPlugins as plugin (plugin.name)}
				<PluginCard {plugin} {usageLoaded} />
			{/each}
		</div>
	{/if}
</div>

<style>
	/* Hero — flat paper with hairline rule */
	.plugins-hero {
		padding: 24px 0 28px;
		border-top: 1px solid var(--border);
		border-bottom: 1px solid var(--border);
	}

	.plugins-hero__eyebrow {
		display: inline-flex;
		align-items: center;
		gap: 8px;
		margin-bottom: 18px;
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.18em;
		text-transform: uppercase;
		color: var(--text-muted);
		font-weight: 500;
	}

	.plugins-hero__eyebrow .dot {
		width: 6px;
		height: 6px;
		border-radius: 50%;
		background: var(--nav-violet);
		flex-shrink: 0;
	}

	/* Filters row */
	.plugins-filters {
		display: flex;
		flex-direction: column;
		align-items: stretch;
		justify-content: space-between;
		gap: 12px;
	}

	@media (min-width: 640px) {
		.plugins-filters {
			flex-direction: row;
			align-items: center;
		}
	}

	.plugins-filter-group {
		display: inline-flex;
		align-items: center;
		gap: 4px;
		padding: 3px;
		background: var(--bg-subtle);
		border: 1px solid var(--border);
		border-radius: var(--radius-md);
	}

	.plugins-filter-chip {
		display: inline-flex;
		align-items: center;
		gap: 8px;
		padding: 5px 12px;
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.16em;
		text-transform: uppercase;
		font-weight: 500;
		color: var(--text-muted);
		background: transparent;
		border: 0;
		border-radius: var(--radius-sm);
		cursor: pointer;
		transition:
			background-color var(--duration-fast) var(--ease),
			color var(--duration-fast) var(--ease);
	}

	.plugins-filter-chip:hover {
		color: var(--text-secondary);
	}

	.plugins-filter-chip.is-active {
		background: var(--bg-base);
		color: var(--text-primary);
		box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
	}

	.plugins-filter-count {
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0;
		font-variant-numeric: tabular-nums;
		color: var(--text-faint);
	}

	.plugins-filter-chip.is-active .plugins-filter-count {
		color: var(--text-muted);
	}

	.plugins-filter-chip:focus-visible {
		outline: 2px solid var(--accent);
		outline-offset: 2px;
	}

	/* Search input */
	.plugins-search {
		position: relative;
		width: 100%;
	}

	@media (min-width: 640px) {
		.plugins-search {
			width: 280px;
		}
	}

	:global(.plugins-search__icon) {
		position: absolute;
		left: 10px;
		top: 50%;
		transform: translateY(-50%);
		color: var(--text-muted);
		pointer-events: none;
	}

	.plugins-search__input {
		width: 100%;
		padding: 8px 12px 8px 32px;
		font-family: var(--font-sans);
		font-size: 14px;
		color: var(--text-primary);
		background: var(--bg-base);
		border: 1px solid var(--border);
		border-radius: var(--radius-md);
		transition:
			border-color var(--duration-fast) var(--ease),
			box-shadow var(--duration-fast) var(--ease);
	}

	.plugins-search__input::placeholder {
		color: var(--text-faint);
	}

	.plugins-search__input:focus {
		outline: none;
		border-color: var(--accent);
		box-shadow: 0 0 0 3px var(--accent-subtle);
	}

	/* Empty / error state — editorial dashed paper */
	.plugins-empty {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		padding: 64px 24px;
		text-align: center;
		background: var(--bg-subtle);
		border: 1px dashed var(--border);
		border-radius: var(--radius-md);
	}

	:global(.plugins-empty__icon) {
		color: var(--text-muted);
		margin-bottom: 14px;
	}

	.plugins-empty__title {
		font-family: var(--font-serif);
		font-style: italic;
		font-size: 20px;
		color: var(--text-primary);
		margin: 0 0 6px;
	}

	.plugins-empty__desc {
		font-size: 13px;
		color: var(--text-muted);
		margin: 0;
	}
</style>
