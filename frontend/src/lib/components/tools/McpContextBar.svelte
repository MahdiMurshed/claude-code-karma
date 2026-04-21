<script lang="ts">
	interface Props {
		mainCalls: number;
		subagentCalls: number;
		compact?: boolean;
		accentColor?: string;
	}

	let { mainCalls, subagentCalls, compact = false, accentColor = 'var(--nav-teal)' }: Props = $props();

	let total = $derived(mainCalls + subagentCalls);
	let mainPct = $derived(total > 0 ? Math.round((mainCalls / total) * 100) : 0);
	let subPct = $derived(total > 0 ? 100 - mainPct : 0);
</script>

{#if total > 0}
	<div
		class="flex items-center gap-2 tabular-nums text-[var(--text-muted)] {compact
			? 'text-[10px]'
			: 'text-xs'}"
	>
		<span class="inline-flex items-center gap-1.5">
			<span
				class="inline-block w-1.5 h-1.5 rounded-full"
				style="background-color: {accentColor};"
			></span>
			<span>main {mainPct}%</span>
		</span>
		<span class="inline-flex items-center gap-1.5">
			<span
				class="inline-block w-1.5 h-1.5 rounded-full"
				style="background-color: var(--text-faint);"
			></span>
			<span>sub {subPct}%</span>
		</span>
	</div>
{/if}
