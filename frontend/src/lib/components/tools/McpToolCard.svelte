<script lang="ts">
	import { ChevronRight } from 'lucide-svelte';
	import McpContextBar from './McpContextBar.svelte';
	import type { McpToolSummary } from '$lib/api-types';
	import TierBadge from '$lib/components/ui/TierBadge.svelte';
	import { getUsageTier } from '$lib/utils';

	interface Props {
		tool: McpToolSummary;
		serverTotalCalls: number;
		maxCalls?: number;
		accentColor?: string;
		serverLabel?: string;
	}

	let {
		tool,
		serverTotalCalls,
		maxCalls = 100,
		accentColor = 'var(--nav-teal)',
		serverLabel
	}: Props = $props();

	let proportion = $derived(serverTotalCalls > 0 ? (tool.calls / serverTotalCalls) * 100 : 0);
	let tier = $derived(getUsageTier(tool.calls, maxCalls));
</script>

<div
	class="
		group
		relative overflow-hidden
		bg-[var(--bg-base)]
		border border-[var(--border)]
		rounded-[var(--radius-md)]
		p-4
		hover:border-[var(--accent)]
		transition-colors duration-200
	"
>
	<!-- Eyebrow: server name, mono-caps with dot marker -->
	{#if serverLabel}
		<div class="flex items-center gap-1.5 mb-1.5 font-mono text-[10px] uppercase tracking-widest text-[var(--text-muted)] truncate">
			<span
				class="inline-block w-1.5 h-1.5 rounded-full flex-shrink-0"
				style="background-color: {accentColor};"
			></span>
			<span class="truncate">{serverLabel}</span>
		</div>
	{/if}

	<!-- Tool Name -->
	<div class="flex items-center justify-between mb-2">
		<h4
			class="text-sm font-semibold text-[var(--text-primary)] truncate pr-2"
			title={tool.full_name}
		>
			{tool.name}
		</h4>
		<ChevronRight
			size={14}
			strokeWidth={1.75}
			class="text-[var(--text-faint)] group-hover:text-[var(--accent)] transition-colors flex-shrink-0"
		/>
	</div>

	<!-- Tier Badge -->
	{#if tier !== 'low'}
		<div class="mb-2">
			<TierBadge {tier} />
		</div>
	{/if}

	<!-- Call Count -->
	<p class="text-xs text-[var(--text-muted)] mb-3 tabular-nums">
		{tool.calls.toLocaleString()} call{tool.calls !== 1 ? 's' : ''}
	</p>

	<!-- Proportion Bar -->
	<div class="h-1 bg-[var(--bg-subtle)] rounded-full overflow-hidden mb-3">
		<div
			class="h-full rounded-full transition-all duration-300"
			style="width: {proportion}%; background-color: {accentColor}; opacity: 0.75;"
		></div>
	</div>

	<!-- Main/Subagent Split -->
	<McpContextBar mainCalls={tool.main_calls} subagentCalls={tool.subagent_calls} compact />
</div>
