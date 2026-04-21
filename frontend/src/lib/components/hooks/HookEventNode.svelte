<script lang="ts">
	import { ChevronRight, AlertCircle } from 'lucide-svelte';
	import { Collapsible } from 'bits-ui';
	import type { HookEventSummary } from '$lib/api-types';
	import HookRegistrationCard from './HookRegistrationCard.svelte';

	interface Props {
		event: HookEventSummary;
		open?: boolean;
		onToggle?: () => void;
	}

	let { event, open = $bindable(false), onToggle }: Props = $props();

	function handleOpenChange(isOpen: boolean) {
		open = isOpen;
		onToggle?.();
	}

	// Derive a short "kind" label for the eyebrow from the event type.
	// e.g. PreToolUse -> PRE, PostToolUse -> POST, PermissionRequest -> PERMISSION.
	function eventKind(name: string): string {
		if (/^Pre[A-Z]/.test(name)) return 'pre';
		if (/^Post[A-Z]/.test(name)) return 'post';
		if (name.startsWith('Permission')) return 'permission';
		if (name.startsWith('Notification')) return 'notify';
		if (name.startsWith('Session')) return 'session';
		if (name.startsWith('Subagent')) return 'subagent';
		if (name === 'Stop' || name.startsWith('Stop')) return 'stop';
		if (name === 'UserPromptSubmit') return 'prompt';
		return 'event';
	}

	let kindLabel = $derived(eventKind(event.event_type));

	// Rail / node tint: error for blocking hooks, nav-orange otherwise.
	let nodeInk = $derived(event.can_block ? 'var(--error)' : 'var(--nav-orange)');
</script>

<Collapsible.Root {open} onOpenChange={handleOpenChange} class="group">
	<div class="relative">
		<!-- Timeline vertical rail (connects nodes) -->
		<div
			class="absolute top-0 bottom-0 w-px"
			style="left: 3px; background: var(--border);"
		></div>

		<!-- Timeline Node Content -->
		<div class="relative pl-8 pb-4">
			<!-- Dot at left rail -->
			<div
				class="absolute left-0 top-[14px] w-[7px] h-[7px] rounded-full z-10"
				style="background: {nodeInk}; box-shadow: 0 0 0 2px var(--bg-base);"
			></div>

			<!-- Collapsible Card -->
			<Collapsible.Trigger class="hook-event-card">
				<!-- Chevron Icon -->
				<ChevronRight
					size={14}
					strokeWidth={1.75}
					class="
						text-[var(--text-muted)]
						transition-transform shrink-0
						{open ? 'rotate-90' : 'rotate-0'}
					"
					style="transition-duration: var(--duration-normal);"
				/>

				<!-- Event Content -->
				<div class="flex-1 min-w-0 text-left">
					<div class="hook-event-eyebrow">
						<span>{kindLabel}</span>
						{#if event.phase}
							<span class="hook-event-eyebrow__sep" aria-hidden="true">/</span>
							<span>{event.phase}</span>
						{/if}
					</div>
					<div class="flex items-center gap-2 mt-0.5">
						<a
							href="/hooks/{encodeURIComponent(event.event_type)}"
							class="hook-event-name"
							onclick={(e) => e.stopPropagation()}
						>
							{event.event_type}
						</a>
						{#if event.can_block}
							<span
								class="hook-event-block-pill"
								title="This hook can block execution"
							>
								<AlertCircle size={9} strokeWidth={2} />
								<span>Can Block</span>
							</span>
						{/if}
					</div>
				</div>

				<!-- Registration Count Badge -->
				<div class="hook-event-count tabular-nums">
					<span class="hook-event-count__num">{event.total_registrations}</span>
					<span class="hook-event-count__label">
						hook{event.total_registrations !== 1 ? 's' : ''}
					</span>
				</div>
			</Collapsible.Trigger>

			<!-- Expanded Content -->
			<Collapsible.Content
				class="overflow-hidden transition-all"
				style="transition-duration: var(--duration-normal);"
			>
				<div class="mt-3 space-y-2 pl-6">
					{#each event.registrations as registration, i (`${registration.source_id}-${i}`)}
						<HookRegistrationCard {registration} />
					{/each}
				</div>
			</Collapsible.Content>
		</div>
	</div>
</Collapsible.Root>

<style>
	/* Ensure smooth transitions for bits-ui Collapsible */
	:global([data-collapsible-content]) {
		transition: height var(--duration-normal) ease-in-out;
	}

	:global(.hook-event-card) {
		display: flex;
		align-items: center;
		gap: 12px;
		width: 100%;
		padding: 10px 14px;
		text-align: left;
		background: var(--bg-base);
		border: 1px solid var(--border);
		border-radius: var(--radius-md);
		transition:
			background var(--duration-fast) var(--ease),
			border-color var(--duration-fast) var(--ease);
	}

	:global(.hook-event-card:hover) {
		background: var(--bg-subtle);
		border-color: var(--border-hover);
	}

	:global(.hook-event-card:focus-visible) {
		outline: none;
	}

	.hook-event-eyebrow {
		display: inline-flex;
		align-items: center;
		gap: 6px;
		font-family: var(--font-mono);
		font-size: 10px;
		font-weight: 500;
		letter-spacing: 0.16em;
		text-transform: uppercase;
		color: var(--text-muted);
	}

	.hook-event-eyebrow__sep {
		color: var(--text-faint);
		font-weight: 400;
	}

	.hook-event-name {
		font-size: 14px;
		font-weight: 600;
		color: var(--text-primary);
		letter-spacing: -0.01em;
		transition: color var(--duration-fast) var(--ease);
	}

	.hook-event-name:hover {
		color: var(--accent);
	}

	.hook-event-block-pill {
		display: inline-flex;
		align-items: center;
		gap: 4px;
		padding: 1px 7px 2px;
		font-family: var(--font-mono);
		font-size: 10px;
		font-weight: 600;
		letter-spacing: 0.14em;
		text-transform: uppercase;
		background: var(--error-subtle);
		color: var(--error);
		border-radius: var(--radius-xs);
	}

	.hook-event-count {
		display: inline-flex;
		align-items: baseline;
		gap: 4px;
		padding: 3px 10px;
		background: var(--bg-muted);
		border-radius: 999px;
		flex-shrink: 0;
	}

	.hook-event-count__num {
		font-family: var(--font-mono);
		font-size: 12px;
		font-weight: 600;
		color: var(--text-primary);
	}

	.hook-event-count__label {
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.12em;
		text-transform: uppercase;
		color: var(--text-muted);
	}
</style>
