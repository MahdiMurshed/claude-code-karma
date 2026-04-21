<script lang="ts">
	import {
		Bot,
		Zap,
		Terminal,
		Wrench,
		Webhook,
		ChevronDown,
		ChevronRight
	} from 'lucide-svelte';
	import type { PluginCapabilities } from '$lib/api-types';

	interface Props {
		capabilities: PluginCapabilities;
		pluginName: string;
		pluginColor?: string;
		pluginColorSubtle?: string;
	}

	let { capabilities, pluginName, pluginColor, pluginColorSubtle }: Props = $props();

	// Strip registry qualifier (e.g., "feature-dev@claude-plugins-official" -> "feature-dev")
	// to match the subagent_type stored in the database
	let pluginShortName = $derived(pluginName.split('@')[0]);

	// Track expanded sections
	let expandedSections = $state<Set<string>>(new Set(['agents', 'skills', 'commands']));

	function toggleSection(section: string) {
		if (expandedSections.has(section)) {
			expandedSections.delete(section);
		} else {
			expandedSections.add(section);
		}
		expandedSections = new Set(expandedSections);
	}

	// Human-readable display name for MCP server identifiers
	// e.g. "plugin_playwright_playwright" -> "Playwright"
	function mcpDisplayName(name: string): string {
		let n = name;
		if (n.startsWith('plugin_')) {
			const parts = n.slice(7).split('_');
			if (parts.length >= 2 && parts[0] === parts[1]) n = parts[0];
			else n = parts.join('_');
		}
		return n.replace(/[-_]/g, ' ').replace(/\b\w/g, (c) => c.toUpperCase());
	}

	// Capability sections config - use $derived for reactivity
	let sections = $derived([
		{
			key: 'agents',
			label: 'Agents',
			items: capabilities.agents,
			linkPrefix: '/agents/',
			requiresPlugin: true
		},
		{
			key: 'skills',
			label: 'Skills',
			items: capabilities.skills,
			linkPrefix: '/skills/',
			requiresPlugin: true
		},
		{
			key: 'commands',
			label: 'Commands',
			items: capabilities.commands,
			linkPrefix: '/commands/',
			requiresPlugin: true
		},
		{
			key: 'mcp_tools',
			label: 'MCP Tools',
			items: capabilities.mcp_tools,
			linkPrefix: '/tools/' as string | null,
			requiresPlugin: false
		},
		{
			key: 'hooks',
			label: 'Hooks',
			items: capabilities.hooks,
			linkPrefix: null as string | null,
			requiresPlugin: false
		}
	]);

	// Total capabilities count
	let totalCount = $derived(
		capabilities.agents.length +
			capabilities.skills.length +
			capabilities.commands.length +
			capabilities.mcp_tools.length +
			capabilities.hooks.length
	);
</script>

<div class="pc-root">
	<div class="pc-header">
		<div class="pc-header__title">
			<span
				class="pc-header__dot"
				style={pluginColor ? `background: ${pluginColor};` : ''}
			></span>
			<h3>What this plugin provides</h3>
		</div>
		<span class="pc-header__total">
			<span class="num">{totalCount}</span>
			<span>total</span>
		</span>
	</div>

	<div class="pc-sections">
		{#each sections as section (section.key)}
			{@const isExpanded = expandedSections.has(section.key)}
			{@const hasItems = section.items.length > 0}

			<div class="pc-section" class:is-empty={!hasItems}>
				<!-- Section header -->
				<button
					onclick={() => toggleSection(section.key)}
					class="pc-section__head"
					disabled={!hasItems}
					aria-expanded={isExpanded}
				>
					<div class="pc-section__label">
						{#if section.key === 'agents'}
							<Bot
								size={14}
								strokeWidth={1.75}
								style={pluginColor ? `color: ${pluginColor};` : ''}
								class={pluginColor ? '' : 'text-[var(--text-muted)]'}
							/>
						{:else if section.key === 'skills'}
							<Zap
								size={14}
								strokeWidth={1.75}
								style={pluginColor ? `color: ${pluginColor};` : ''}
								class={pluginColor ? '' : 'text-[var(--text-muted)]'}
							/>
						{:else if section.key === 'commands'}
							<Terminal
								size={14}
								strokeWidth={1.75}
								style={pluginColor ? `color: ${pluginColor};` : ''}
								class={pluginColor ? '' : 'text-[var(--text-muted)]'}
							/>
						{:else if section.key === 'mcp_tools'}
							<Wrench
								size={14}
								strokeWidth={1.75}
								style={pluginColor ? `color: ${pluginColor};` : ''}
								class={pluginColor ? '' : 'text-[var(--text-muted)]'}
							/>
						{:else}
							<Webhook
								size={14}
								strokeWidth={1.75}
								style={pluginColor ? `color: ${pluginColor};` : ''}
								class={pluginColor ? '' : 'text-[var(--text-muted)]'}
							/>
						{/if}
						<span class="pc-section__name">{section.label}</span>
						<span class="pc-section__count">
							<span class="num">{section.items.length}</span>
						</span>
					</div>
					{#if hasItems}
						{#if isExpanded}
							<ChevronDown size={14} strokeWidth={1.75} class="text-[var(--text-muted)]" />
						{:else}
							<ChevronRight size={14} strokeWidth={1.75} class="text-[var(--text-muted)]" />
						{/if}
					{/if}
				</button>

				<!-- Section content -->
				{#if isExpanded && hasItems}
					<div class="pc-section__body">
						<div class="pc-chips">
							{#each section.items as item}
								{#if section.linkPrefix}
									{@const fullName = section.requiresPlugin
										? pluginShortName + ':' + item
										: item}
									{@const displayName =
										section.key === 'mcp_tools' ? mcpDisplayName(item) : item}
									<a
										href="{section.linkPrefix}{encodeURIComponent(fullName)}"
										class="pc-chip"
										style="background-color: {pluginColorSubtle ||
											'var(--bg-subtle)'}; color: {pluginColor ||
											'var(--text-secondary)'};"
										onmouseenter={(e) => {
											e.currentTarget.style.borderColor =
												pluginColor || 'var(--border-hover)';
										}}
										onmouseleave={(e) => {
											e.currentTarget.style.borderColor = 'transparent';
										}}
									>
										{#if section.key === 'commands'}
											<Terminal size={11} strokeWidth={1.75} class="flex-shrink-0 opacity-70" />
										{/if}
										{displayName}
									</a>
								{:else}
									<span
										class="pc-chip pc-chip--static"
										style="background-color: {pluginColorSubtle ||
											'var(--bg-subtle)'}; color: {pluginColor ||
											'var(--text-secondary)'};"
									>
										{item}
									</span>
								{/if}
							{/each}
						</div>
					</div>
				{/if}
			</div>
		{/each}
	</div>
</div>

<style>
	.pc-root {
		display: flex;
		flex-direction: column;
		gap: 0;
	}

	.pc-header {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 12px;
		margin-bottom: 14px;
	}

	.pc-header__title {
		display: inline-flex;
		align-items: center;
		gap: 8px;
	}

	.pc-header__dot {
		width: 6px;
		height: 6px;
		border-radius: 50%;
		background: var(--accent);
		flex-shrink: 0;
	}

	.pc-header h3 {
		font-family: var(--font-mono);
		font-size: 11px;
		letter-spacing: 0.16em;
		text-transform: uppercase;
		font-weight: 500;
		color: var(--text-primary);
		margin: 0;
	}

	.pc-header__total {
		display: inline-flex;
		align-items: baseline;
		gap: 5px;
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.14em;
		text-transform: uppercase;
		color: var(--text-muted);
	}

	.pc-header__total .num {
		color: var(--text-primary);
		font-variant-numeric: tabular-nums;
		letter-spacing: 0.04em;
	}

	.pc-sections {
		display: flex;
		flex-direction: column;
		gap: 8px;
	}

	.pc-section {
		border: 1px solid var(--border);
		border-radius: var(--radius-md);
		overflow: hidden;
		background: var(--bg-base);
		transition: border-color var(--duration-fast) var(--ease);
	}

	.pc-section:hover:not(.is-empty) {
		border-color: var(--border-hover);
	}

	.pc-section__head {
		width: 100%;
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 10px 14px;
		background: var(--bg-subtle);
		border: 0;
		text-align: left;
		cursor: pointer;
		transition: background-color var(--duration-fast) var(--ease);
	}

	.pc-section__head:hover:not(:disabled) {
		background: var(--bg-muted);
	}

	.pc-section__head:disabled {
		cursor: default;
		opacity: 0.6;
	}

	.pc-section__head:focus-visible {
		outline: 2px solid var(--accent);
		outline-offset: -2px;
	}

	.pc-section__label {
		display: inline-flex;
		align-items: center;
		gap: 10px;
	}

	.pc-section__name {
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.18em;
		text-transform: uppercase;
		font-weight: 500;
		color: var(--text-primary);
	}

	.pc-section__count {
		display: inline-flex;
		align-items: baseline;
		padding: 1px 6px;
		background: var(--bg-base);
		border: 1px solid var(--border-subtle);
		border-radius: var(--radius-xs);
		font-family: var(--font-mono);
		font-size: 10px;
		color: var(--text-muted);
	}

	.pc-section__count .num {
		color: var(--text-secondary);
		font-variant-numeric: tabular-nums;
	}

	.pc-section__body {
		padding: 12px 14px 14px;
		background: var(--bg-base);
		border-top: 1px solid var(--border-subtle);
	}

	.pc-chips {
		display: flex;
		flex-wrap: wrap;
		gap: 6px;
	}

	.pc-chip {
		display: inline-flex;
		align-items: center;
		gap: 6px;
		padding: 4px 10px;
		font-family: var(--font-mono);
		font-size: 11px;
		font-weight: 500;
		letter-spacing: 0.02em;
		border-radius: var(--radius-sm);
		border: 1px solid transparent;
		text-decoration: none;
		transition:
			border-color var(--duration-fast) var(--ease),
			transform var(--duration-fast) var(--ease);
	}

	.pc-chip:not(.pc-chip--static):hover {
		transform: translateY(-1px);
	}

	.pc-chip--static {
		cursor: default;
	}

	.pc-chip:focus-visible {
		outline: 2px solid var(--accent);
		outline-offset: 2px;
	}
</style>
