<script lang="ts">
	import type { HookRegistration } from '$lib/api-types';
	import { getHookSourceColorVars } from '$lib/utils';
	import { truncate } from '$lib/utils';

	interface Props {
		registration: HookRegistration;
	}

	let { registration }: Props = $props();

	let sourceColors = $derived(
		getHookSourceColorVars(registration.source_type, registration.source_name)
	);

	// Language display names
	const languageLabels: Record<string, string> = {
		python: 'Python',
		node: 'Node.js',
		shell: 'Shell',
		bash: 'Shell'
	};

	let languageLabel = $derived(
		languageLabels[registration.script_language] || registration.script_language
	);
</script>

<div class="reg-card" style="border-left: 3px solid {sourceColors.color};">
	<!-- Source and Script Info -->
	<div class="flex items-start justify-between gap-3 mb-2">
		<div class="flex-1 min-w-0">
			<div class="flex items-center gap-2 flex-wrap">
				<span class="reg-source">
					{registration.source_name}
				</span>
				{#if registration.script_filename}
					<span class="reg-sep" aria-hidden="true">·</span>
					<span class="reg-script-name" title={registration.script_filename}>
						{registration.script_filename}
					</span>
				{/if}
			</div>
		</div>

		<!-- Language eyebrow -->
		<span class="reg-language">
			{languageLabel}
		</span>
	</div>

	<!-- Matcher (only if not "*") -->
	{#if registration.matcher && registration.matcher !== '*'}
		<div class="reg-matcher">
			<span class="reg-matcher__label">Matcher</span>
			<code class="reg-matcher__value">{registration.matcher}</code>
		</div>
	{/if}

	<!-- Command -->
	<div class="reg-command" title={registration.command}>
		{truncate(registration.command, 80)}
	</div>
</div>

<style>
	.reg-card {
		position: relative;
		padding: 10px 12px;
		background: var(--bg-base);
		border: 1px solid var(--border);
		border-radius: var(--radius-md);
		overflow: hidden;
		transition: border-color var(--duration-fast) var(--ease);
	}

	.reg-card:hover {
		border-color: var(--border-hover);
	}

	.reg-source {
		font-size: 12px;
		font-weight: 600;
		color: var(--text-primary);
		letter-spacing: -0.005em;
	}

	.reg-sep {
		color: var(--text-faint);
		font-size: 11px;
	}

	.reg-script-name {
		font-family: var(--font-mono);
		font-size: 11px;
		color: var(--text-muted);
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		max-width: 240px;
	}

	.reg-language {
		flex-shrink: 0;
		padding: 2px 6px;
		font-family: var(--font-mono);
		font-size: 10px;
		font-weight: 500;
		letter-spacing: 0.14em;
		text-transform: uppercase;
		color: var(--text-muted);
		background: var(--bg-subtle);
		border-radius: var(--radius-xs);
	}

	.reg-matcher {
		display: flex;
		align-items: center;
		gap: 8px;
		margin-bottom: 8px;
	}

	.reg-matcher__label {
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.16em;
		text-transform: uppercase;
		color: var(--text-muted);
	}

	.reg-matcher__value {
		font-family: var(--font-mono);
		font-size: 11px;
		color: var(--text-secondary);
	}

	.reg-command {
		padding: 7px 10px;
		font-family: var(--font-mono);
		font-size: 11px;
		color: var(--text-secondary);
		background: var(--bg-subtle);
		border: 1px solid var(--border-subtle);
		border-radius: var(--radius-sm);
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}
</style>
