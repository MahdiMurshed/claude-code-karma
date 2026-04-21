<script lang="ts">
	import type { McpServer, McpToolSummary } from '$lib/api-types';
	import { ChevronDown, ChevronUp } from 'lucide-svelte';
	import { getServerColorVars } from '$lib/utils/mcp';

	interface FlatTool extends McpToolSummary {
		serverName: string;
		serverDisplayName: string;
		lastUsed: string | null;
		pluginName: string | null;
	}

	interface Props {
		servers: McpServer[];
	}

	let { servers }: Props = $props();

	// Flatten all tools into a single list
	let allTools = $derived.by<FlatTool[]>(() => {
		const tools: FlatTool[] = [];
		for (const server of servers) {
			for (const tool of server.tools) {
				tools.push({
					...tool,
					serverName: server.name,
					serverDisplayName: server.display_name,
					lastUsed: server.last_used,
					pluginName: server.plugin_name ?? null
				});
			}
		}
		return tools;
	});

	type SortKey = 'calls' | 'name' | 'server' | 'sessions' | 'main' | 'sub';

	// Sort state
	let sortKey = $state<SortKey>('calls');
	let sortDir = $state<'asc' | 'desc'>('desc');

	let sortedTools = $derived.by(() => {
		const sorted = [...allTools];
		sorted.sort((a, b) => {
			let cmp = 0;
			switch (sortKey) {
				case 'calls':
					cmp = a.calls - b.calls;
					break;
				case 'name':
					cmp = a.name.localeCompare(b.name);
					break;
				case 'server':
					cmp = a.serverDisplayName.localeCompare(b.serverDisplayName);
					break;
				case 'sessions':
					cmp = a.session_count - b.session_count;
					break;
				case 'main':
					cmp =
						(a.calls > 0 ? a.main_calls / a.calls : 0) -
						(b.calls > 0 ? b.main_calls / b.calls : 0);
					break;
				case 'sub':
					cmp =
						(a.calls > 0 ? a.subagent_calls / a.calls : 0) -
						(b.calls > 0 ? b.subagent_calls / b.calls : 0);
					break;
			}
			return sortDir === 'desc' ? -cmp : cmp;
		});
		return sorted;
	});

	function toggleSort(key: SortKey) {
		if (sortKey === key) {
			sortDir = sortDir === 'desc' ? 'asc' : 'desc';
		} else {
			sortKey = key;
			sortDir = 'desc';
		}
	}

	function ariaSortFor(key: SortKey): 'ascending' | 'descending' | 'none' {
		if (sortKey !== key) return 'none';
		return sortDir === 'desc' ? 'descending' : 'ascending';
	}
</script>

<div
	class="overflow-x-auto border border-[var(--border)] rounded-[var(--radius-md)] bg-[var(--bg-base)]"
>
	<table class="w-full text-sm tool-table">
		<thead>
			<tr class="border-b border-[var(--border)]">
				<th
					class="text-left px-4 py-3 th-editorial"
					aria-sort={ariaSortFor('name')}
				>
					<button type="button" onclick={() => toggleSort('name')} class="th-btn">
						<span>Tool</span>
						{#if sortKey === 'name'}
							{#if sortDir === 'desc'}
								<ChevronDown size={12} strokeWidth={1.75} />
							{:else}
								<ChevronUp size={12} strokeWidth={1.75} />
							{/if}
						{/if}
					</button>
				</th>
				<th
					class="text-left px-4 py-3 th-editorial"
					aria-sort={ariaSortFor('server')}
				>
					<button type="button" onclick={() => toggleSort('server')} class="th-btn">
						<span>Server</span>
						{#if sortKey === 'server'}
							{#if sortDir === 'desc'}
								<ChevronDown size={12} strokeWidth={1.75} />
							{:else}
								<ChevronUp size={12} strokeWidth={1.75} />
							{/if}
						{/if}
					</button>
				</th>
				<th
					class="text-right px-4 py-3 th-editorial"
					aria-sort={ariaSortFor('calls')}
				>
					<button
						type="button"
						onclick={() => toggleSort('calls')}
						class="th-btn th-btn--right"
					>
						{#if sortKey === 'calls'}
							{#if sortDir === 'desc'}
								<ChevronDown size={12} strokeWidth={1.75} />
							{:else}
								<ChevronUp size={12} strokeWidth={1.75} />
							{/if}
						{/if}
						<span>Calls</span>
					</button>
				</th>
				<th
					class="text-right px-4 py-3 th-editorial hidden md:table-cell"
					aria-sort={ariaSortFor('sessions')}
				>
					<button
						type="button"
						onclick={() => toggleSort('sessions')}
						class="th-btn th-btn--right"
					>
						{#if sortKey === 'sessions'}
							{#if sortDir === 'desc'}
								<ChevronDown size={12} strokeWidth={1.75} />
							{:else}
								<ChevronUp size={12} strokeWidth={1.75} />
							{/if}
						{/if}
						<span>Sessions</span>
					</button>
				</th>
				<th
					class="text-right px-4 py-3 th-editorial hidden lg:table-cell"
					aria-sort={ariaSortFor('main')}
				>
					<button
						type="button"
						onclick={() => toggleSort('main')}
						class="th-btn th-btn--right"
					>
						{#if sortKey === 'main'}
							{#if sortDir === 'desc'}
								<ChevronDown size={12} strokeWidth={1.75} />
							{:else}
								<ChevronUp size={12} strokeWidth={1.75} />
							{/if}
						{/if}
						<span>Main %</span>
					</button>
				</th>
				<th
					class="text-right px-4 py-3 th-editorial hidden lg:table-cell"
					aria-sort={ariaSortFor('sub')}
				>
					<button
						type="button"
						onclick={() => toggleSort('sub')}
						class="th-btn th-btn--right"
					>
						{#if sortKey === 'sub'}
							{#if sortDir === 'desc'}
								<ChevronDown size={12} strokeWidth={1.75} />
							{:else}
								<ChevronUp size={12} strokeWidth={1.75} />
							{/if}
						{/if}
						<span>Sub %</span>
					</button>
				</th>
			</tr>
		</thead>
		<tbody>
			{#each sortedTools as tool (tool.full_name)}
				{@const toolColor = getServerColorVars(tool.serverName, tool.pluginName)}
				<tr
					class="border-b border-[var(--border-subtle)] last:border-b-0 hover:bg-[var(--bg-subtle)] transition-colors"
				>
					<td class="px-4 py-3">
						<div class="flex items-center gap-2.5">
							<span
								class="w-1.5 h-1.5 rounded-full flex-shrink-0"
								style="background-color: {toolColor.color};"
							></span>
							<a
								href="/tools/{encodeURIComponent(tool.serverName)}/{encodeURIComponent(tool.name)}"
								class="font-medium text-[var(--text-primary)] hover:text-[var(--accent)] transition-colors"
							>
								{tool.name}
							</a>
						</div>
					</td>
					<td class="px-4 py-3">
						<a
							href="/tools/{tool.serverName}"
							class="text-[var(--text-secondary)] hover:text-[var(--accent)] transition-colors"
						>
							{tool.serverDisplayName}
						</a>
					</td>
					<td class="px-4 py-3 text-right tabular-nums text-[var(--text-primary)] font-medium">
						{tool.calls.toLocaleString()}
					</td>
					<td
						class="px-4 py-3 text-right tabular-nums text-[var(--text-muted)] hidden md:table-cell"
					>
						{tool.session_count}
					</td>
					<td
						class="px-4 py-3 text-right tabular-nums text-[var(--text-muted)] hidden lg:table-cell"
					>
						{tool.calls > 0 ? Math.round((tool.main_calls / tool.calls) * 100) : 0}%
					</td>
					<td
						class="px-4 py-3 text-right tabular-nums text-[var(--text-muted)] hidden lg:table-cell"
					>
						{tool.calls > 0 ? Math.round((tool.subagent_calls / tool.calls) * 100) : 0}%
					</td>
				</tr>
			{/each}
		</tbody>
	</table>
</div>

<style>
	.th-editorial {
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.14em;
		text-transform: uppercase;
		font-weight: 500;
		color: var(--text-muted);
	}

	.th-btn {
		display: inline-flex;
		align-items: center;
		gap: 4px;
		color: inherit;
		transition: color var(--duration-fast, 150ms) ease;
	}

	.th-btn--right {
		justify-content: flex-end;
		width: 100%;
	}

	.th-btn:hover {
		color: var(--text-primary);
	}

	.th-btn:focus-visible {
		outline: 2px solid var(--accent);
		outline-offset: 2px;
		border-radius: var(--radius-xs);
	}
</style>
