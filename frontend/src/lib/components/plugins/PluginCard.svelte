<script lang="ts">
	import { Puzzle } from 'lucide-svelte';
	import { formatCost } from '$lib/utils';
	import type { PluginSummary } from '$lib/api-types';

	interface Props {
		plugin: PluginSummary;
		usageLoaded?: boolean;
		class?: string;
	}

	let { plugin, usageLoaded = true, class: className = '' }: Props = $props();

	// Format days since update — editorial copy
	let updateLabel = $derived(
		plugin.days_since_update === 0
			? 'Updated today'
			: plugin.days_since_update === 1
				? 'Updated yesterday'
				: `Updated ${plugin.days_since_update}d ago`
	);

	let detailHref = $derived(`/plugins/${encodeURIComponent(plugin.name)}`);

	let displayName = $derived(plugin.name.split('@')[0]);
</script>

<a href={detailHref} class="plugin-card {className}" data-list-item>
	<!-- Header: ink box + optional official pill -->
	<div class="plugin-card__head">
		<div class="plugin-card__icon" aria-hidden="true">
			<Puzzle size={16} strokeWidth={1.75} />
		</div>
		{#if plugin.is_official}
			<span class="plugin-card__badge">Official</span>
		{/if}
	</div>

	<!-- Plugin name — serif italic display voice -->
	<h3 class="plugin-card__name" title={displayName}>
		{displayName}
	</h3>

	<!-- Description -->
	{#if plugin.description}
		<p class="plugin-card__desc">
			{plugin.description}
		</p>
	{:else}
		<p class="plugin-card__desc plugin-card__desc--empty">No description provided.</p>
	{/if}

	<!-- Metadata eyebrow — mono caps, tabular nums, dot separators -->
	<div class="plugin-card__meta">
		<span class="plugin-card__meta-item">
			<span class="num">{plugin.agent_count}</span>
			<span>agents</span>
		</span>
		<span class="dot-sep" aria-hidden="true"></span>
		<span class="plugin-card__meta-item">
			<span class="num">{plugin.skill_count}</span>
			<span>skills</span>
		</span>
		<span class="dot-sep" aria-hidden="true"></span>
		<span class="plugin-card__meta-item">
			<span class="num">{plugin.command_count}</span>
			<span>cmds</span>
		</span>
	</div>

	<!-- Footer: runs/cost on left, version/update on right -->
	<div class="plugin-card__foot">
		<div class="plugin-card__usage">
			{#if usageLoaded}
				<span class="plugin-card__usage-item">
					<span class="num">{plugin.total_runs.toLocaleString()}</span>
					<span class="plugin-card__usage-label">runs</span>
				</span>
				<span class="dot-sep" aria-hidden="true"></span>
				<span class="plugin-card__usage-item">
					<span class="num">{formatCost(plugin.estimated_cost_usd)}</span>
				</span>
			{:else}
				<span class="plugin-card__skeleton"></span>
				<span class="dot-sep" aria-hidden="true"></span>
				<span class="plugin-card__skeleton plugin-card__skeleton--short"></span>
			{/if}
		</div>
		<div class="plugin-card__version">
			<span>{updateLabel}</span>
			<span class="dot-sep" aria-hidden="true"></span>
			<span class="plugin-card__version-tag">v{plugin.latest_version.substring(0, 7)}</span>
		</div>
	</div>
</a>

<style>
	.plugin-card {
		display: flex;
		flex-direction: column;
		gap: 0;
		padding: 18px 20px 16px;
		background: var(--bg-base);
		border: 1px solid var(--border);
		border-radius: var(--radius-md);
		text-decoration: none;
		color: inherit;
		transition:
			border-color var(--duration-fast) var(--ease),
			background-color var(--duration-fast) var(--ease);
	}

	.plugin-card:hover {
		border-color: var(--accent);
	}

	.plugin-card:focus-visible {
		outline: 2px solid var(--accent);
		outline-offset: 2px;
	}

	/* Header row */
	.plugin-card__head {
		display: flex;
		align-items: center;
		justify-content: space-between;
		margin-bottom: 14px;
	}

	.plugin-card__icon {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		width: 32px;
		height: 32px;
		border-radius: var(--radius-sm);
		background: var(--nav-violet-subtle);
		color: var(--nav-violet);
		flex-shrink: 0;
	}

	.plugin-card__badge {
		display: inline-flex;
		align-items: center;
		padding: 3px 7px;
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.16em;
		text-transform: uppercase;
		font-weight: 500;
		color: var(--accent);
		background: var(--accent-subtle);
		border-radius: var(--radius-xs);
	}

	/* Name — serif italic display voice */
	.plugin-card__name {
		font-family: var(--font-serif);
		font-style: italic;
		font-size: 22px;
		line-height: 1.15;
		letter-spacing: -0.005em;
		color: var(--text-primary);
		margin: 0 0 6px;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	/* Description */
	.plugin-card__desc {
		font-size: 13.5px;
		line-height: 1.5;
		color: var(--text-secondary);
		margin: 0 0 14px;
		display: -webkit-box;
		-webkit-line-clamp: 2;
		-webkit-box-orient: vertical;
		overflow: hidden;
		min-height: 2.8em;
	}

	.plugin-card__desc--empty {
		font-style: italic;
		color: var(--text-faint);
	}

	/* Metadata — mono caps eyebrow with dots */
	.plugin-card__meta {
		display: flex;
		align-items: center;
		flex-wrap: wrap;
		gap: 10px;
		padding: 10px 0;
		margin-bottom: 2px;
		border-top: 1px solid var(--border-subtle);
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.16em;
		text-transform: uppercase;
		color: var(--text-muted);
		font-weight: 500;
	}

	.plugin-card__meta-item {
		display: inline-flex;
		align-items: baseline;
		gap: 5px;
	}

	.plugin-card__meta-item .num,
	.plugin-card__usage-item .num,
	.plugin-card__version .num {
		color: var(--text-primary);
		font-variant-numeric: tabular-nums;
		letter-spacing: 0.04em;
	}

	/* Footer */
	.plugin-card__foot {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 12px;
		padding-top: 10px;
		border-top: 1px solid var(--border-subtle);
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.14em;
		text-transform: uppercase;
		color: var(--text-muted);
		font-weight: 500;
	}

	.plugin-card__usage,
	.plugin-card__version {
		display: inline-flex;
		align-items: center;
		gap: 8px;
		flex-wrap: nowrap;
		min-width: 0;
	}

	.plugin-card__usage-item {
		display: inline-flex;
		align-items: baseline;
		gap: 4px;
	}

	.plugin-card__usage-label {
		color: var(--text-muted);
	}

	.plugin-card__version {
		color: var(--text-faint);
	}

	.plugin-card__version-tag {
		color: var(--text-muted);
		letter-spacing: 0.05em;
	}

	.plugin-card__skeleton {
		display: inline-block;
		width: 48px;
		height: 8px;
		background: var(--bg-muted);
		border-radius: var(--radius-xs);
		animation: plugin-shimmer 1.2s ease-in-out infinite;
	}

	.plugin-card__skeleton--short {
		width: 32px;
	}

	@keyframes plugin-shimmer {
		0%,
		100% {
			opacity: 0.6;
		}
		50% {
			opacity: 1;
		}
	}

	/* Dot separator — 3px neutral dot */
	.dot-sep {
		display: inline-block;
		width: 3px;
		height: 3px;
		border-radius: 50%;
		background: var(--text-faint);
		flex-shrink: 0;
	}
</style>
