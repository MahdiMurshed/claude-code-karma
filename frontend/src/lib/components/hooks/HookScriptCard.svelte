<script lang="ts">
	import { goto } from '$app/navigation';
	import { FileCode, ArrowUpRight } from 'lucide-svelte';
	import type { HookScript } from '$lib/api-types';
	import { getHookSourceColorVars } from '$lib/utils';

	interface Props {
		script: HookScript;
		sourceType: string;
		sourceName: string;
	}

	let { script, sourceType, sourceName }: Props = $props();

	let sourceColors = $derived(getHookSourceColorVars(sourceType, sourceName));

	// Language display names
	const languageLabels: Record<string, string> = {
		python: 'Python',
		node: 'Node.js',
		shell: 'Shell',
		bash: 'Shell'
	};

	let languageLabel = $derived(languageLabels[script.language] || script.language);
</script>

<a
	href="/hooks/scripts/{encodeURIComponent(script.filename)}"
	class="script-card"
	style="border-left: 3px solid {sourceColors.color};"
>
	<!-- Filename and Language -->
	<div class="flex items-start justify-between gap-3 mb-3">
		<div class="flex items-center gap-3 min-w-0">
			<div
				class="script-icon"
				style="background-color: {sourceColors.subtle}; color: {sourceColors.color};"
			>
				<FileCode size={16} strokeWidth={1.75} />
			</div>
			<div class="min-w-0">
				<h3 class="script-title">{script.filename}</h3>
				{#if script.is_symlink && script.symlink_target}
					<p class="script-symlink" title={script.symlink_target}>
						→ {script.symlink_target}
					</p>
				{/if}
			</div>
		</div>

		<!-- Language eyebrow -->
		<span class="script-language">
			{languageLabel}
		</span>
	</div>

	<!-- Event Type Pills -->
	<div class="flex flex-wrap gap-1.5 mb-4">
		{#each script.event_types as eventType}
			<button
				type="button"
				onclick={(e) => {
					e.preventDefault();
					e.stopPropagation();
					goto(`/hooks/${encodeURIComponent(eventType)}`);
				}}
				class="script-event-pill"
			>
				<span>{eventType}</span>
				<ArrowUpRight size={9} strokeWidth={2} />
			</button>
		{/each}
	</div>

	<!-- Stats -->
	<div class="script-stats">
		<span class="tabular-nums">
			<span class="script-stats__num">{script.registrations}</span>
			<span class="script-stats__label">
				registration{script.registrations !== 1 ? 's' : ''}
			</span>
		</span>
		<span class="script-stats__sep" aria-hidden="true">·</span>
		<span class="tabular-nums">
			<span class="script-stats__num">{script.event_types.length}</span>
			<span class="script-stats__label">
				event type{script.event_types.length !== 1 ? 's' : ''}
			</span>
		</span>
	</div>
</a>

<style>
	.script-card {
		display: block;
		padding: 16px;
		background: var(--bg-base);
		border: 1px solid var(--border);
		border-radius: var(--radius-md);
		transition:
			border-color var(--duration-fast) var(--ease),
			background var(--duration-fast) var(--ease);
		position: relative;
		overflow: hidden;
	}

	.script-card:hover {
		border-color: var(--border-hover);
		background: var(--bg-subtle);
	}

	.script-icon {
		display: inline-flex;
		align-items: center;
		justify-content: center;
		width: 32px;
		height: 32px;
		border-radius: var(--radius-sm);
		flex-shrink: 0;
	}

	.script-title {
		font-size: 13px;
		font-weight: 600;
		letter-spacing: -0.01em;
		color: var(--text-primary);
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		transition: color var(--duration-fast) var(--ease);
	}

	.script-card:hover .script-title {
		color: var(--accent);
	}

	.script-symlink {
		margin-top: 2px;
		font-family: var(--font-mono);
		font-size: 10px;
		color: var(--text-muted);
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.script-language {
		flex-shrink: 0;
		padding: 2px 7px;
		font-family: var(--font-mono);
		font-size: 10px;
		font-weight: 500;
		letter-spacing: 0.16em;
		text-transform: uppercase;
		color: var(--text-muted);
		background: var(--bg-subtle);
		border: 1px solid var(--border-subtle);
		border-radius: var(--radius-xs);
	}

	.script-event-pill {
		display: inline-flex;
		align-items: center;
		gap: 4px;
		padding: 2px 7px;
		font-family: var(--font-mono);
		font-size: 10px;
		font-weight: 500;
		letter-spacing: 0.06em;
		color: var(--text-secondary);
		background: var(--bg-muted);
		border: 1px solid transparent;
		border-radius: var(--radius-xs);
		cursor: pointer;
		transition:
			color var(--duration-fast) var(--ease),
			background var(--duration-fast) var(--ease),
			border-color var(--duration-fast) var(--ease);
	}

	.script-event-pill:hover {
		color: var(--nav-orange);
		background: var(--nav-orange-subtle);
		border-color: var(--nav-orange);
	}

	.script-stats {
		display: flex;
		align-items: center;
		gap: 8px;
		padding-top: 12px;
		border-top: 1px solid var(--border-subtle);
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.1em;
		text-transform: uppercase;
		color: var(--text-muted);
	}

	.script-stats__num {
		font-weight: 600;
		color: var(--text-primary);
	}

	.script-stats__label {
		color: var(--text-muted);
	}

	.script-stats__sep {
		color: var(--text-faint);
		text-transform: none;
		letter-spacing: 0;
	}
</style>
