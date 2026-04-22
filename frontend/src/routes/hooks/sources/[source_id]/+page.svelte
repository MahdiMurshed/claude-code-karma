<script lang="ts">
	import {
		FolderOpen,
		Puzzle,
		Code,
		FileCode,
		Link as LinkIcon,
		ExternalLink
	} from 'lucide-svelte';
	import PageHeader from '$lib/components/layout/PageHeader.svelte';
	import Badge from '$lib/components/ui/Badge.svelte';
	import { getHookSourceColorVars } from '$lib/utils';

	let { data } = $props();

	// All 13 hook event types for coverage matrix
	const ALL_EVENT_TYPES = [
		'SessionStart',
		'UserPromptSubmit',
		'PreToolUse',
		'PostToolUse',
		'PostToolUseFailure',
		'SubagentStart',
		'SubagentStop',
		'Stop',
		'PreCompact',
		'PermissionRequest',
		'Notification',
		'SessionEnd',
		'Setup'
	];

	// Color vars based on source type
	let colorVars = $derived(
		getHookSourceColorVars(data.detail.source.source_type, data.detail.source.source_name)
	);

	// Icon based on source type
	let sourceIcon = $derived(
		data.detail.source.source_type === 'plugin'
			? Puzzle
			: data.detail.source.source_type === 'project'
				? Code
				: FolderOpen
	);

	// Get event type color (use hook source color for covered events, muted for uncovered)
	function getEventColor(eventType: string): string {
		return data.detail.coverage_matrix[eventType] ? colorVars.color : 'var(--text-faint)';
	}

	function getEventBg(eventType: string): string {
		return data.detail.coverage_matrix[eventType] ? colorVars.subtle : 'var(--bg-muted)';
	}
</script>

<div class="space-y-8">
	<!-- Plugin Banner (if applicable) -->
	{#if data.detail.source.source_type === 'plugin' && data.detail.source.plugin_id}
		<a
			href="/plugins/{encodeURIComponent(data.detail.source.plugin_id)}"
			class="flex items-center gap-3 px-5 py-4 rounded-[var(--radius-md)] border border-[var(--border)] hover:border-[var(--border-hover)] transition-colors group"
			style="border-left: 3px solid {colorVars.color}; background-color: {colorVars.subtle};"
		>
			<Puzzle size={18} strokeWidth={1.75} style="color: {colorVars.color};" />
			<div class="flex-1">
				<p class="font-mono text-[10px] uppercase tracking-widest text-[var(--text-muted)] mb-1">
					Plugin source
				</p>
				<p class="text-sm text-[var(--text-primary)]">
					<span style="color: {colorVars.color};">{data.detail.source.plugin_id}</span>
					<span class="text-[var(--text-muted)]">— view plugin details</span>
				</p>
			</div>
			<ExternalLink
				size={14}
				strokeWidth={1.75}
				class="text-[var(--text-muted)] group-hover:text-[var(--accent)] transition-colors"
			/>
		</a>
	{/if}

	<!-- Page Header -->
	<PageHeader
		title={data.detail.source.source_name}
		icon={sourceIcon}
		iconColorRaw={colorVars}
		breadcrumbs={[
			{ label: 'Dashboard', href: '/' },
			{ label: 'Hooks', href: '/hooks' },
			{ label: data.detail.source.source_name }
		]}
	>
		{#snippet badges()}
			<Badge variant="accent">
				{data.detail.source.total_registrations} registration{data.detail.source
					.total_registrations !== 1
					? 's'
					: ''}
			</Badge>
			<Badge variant="slate">
				{data.detail.scripts.length} script{data.detail.scripts.length !== 1 ? 's' : ''}
			</Badge>
		{/snippet}
	</PageHeader>

	<!-- Event Coverage Matrix -->
	<div class="bg-[var(--bg-base)] border border-[var(--border)] rounded-[var(--radius-md)] p-6">
		<div class="flex items-center gap-2 mb-6">
			<span
				class="inline-block w-1.5 h-1.5 rounded-full"
				style="background-color: {colorVars.color};"
			></span>
			<h2
				class="font-mono text-[11px] uppercase tracking-widest font-medium text-[var(--text-secondary)]"
			>
				Event coverage
			</h2>
		</div>

		<div class="flex flex-wrap gap-4">
			{#each ALL_EVENT_TYPES as eventType}
				<a
					href="/hooks/{eventType}"
					class="flex flex-col items-center gap-2 group transition-transform hover:scale-105"
				>
					<!-- Dot/Circle -->
					<div
						class="w-11 h-11 rounded-full border flex items-center justify-center transition-colors {data
							.detail.coverage_matrix[eventType]
							? ''
							: 'border-dashed'}"
						style="background-color: {getEventBg(
							eventType
						)}; border-color: {getEventColor(eventType)};"
					>
						{#if data.detail.coverage_matrix[eventType]}
							<div
								class="w-1.5 h-1.5 rounded-full"
								style="background-color: {colorVars.color};"
							></div>
						{/if}
					</div>
					<!-- Label -->
					<span
						class="text-xs font-medium text-center max-w-[80px] leading-tight group-hover:text-[var(--text-primary)] transition-colors"
						style="color: {getEventColor(eventType)};"
					>
						{eventType}
					</span>
				</a>
			{/each}
		</div>

		<!-- Coverage Summary -->
		<div
			class="mt-6 pt-6 border-t border-[var(--border)] flex items-center gap-4 font-mono text-[11px] uppercase tracking-widest text-[var(--text-muted)] tabular-nums"
		>
			<span class="flex items-center gap-1.5">
				<span
					class="inline-block w-1.5 h-1.5 rounded-full"
					style="background-color: {colorVars.color};"
				></span>
				{data.detail.source.event_types_covered.length} of {ALL_EVENT_TYPES.length} covered
			</span>
			{#if data.detail.source.blocking_hooks_count > 0}
				<span class="text-[var(--text-faint)]">·</span>
				<span class="flex items-center gap-1.5 text-[var(--error)]">
					<span
						class="inline-block w-1.5 h-1.5 rounded-full"
						style="background-color: var(--error);"
					></span>
					{data.detail.source.blocking_hooks_count} blocking
				</span>
			{/if}
		</div>
	</div>

	<!-- Scripts Section -->
	<div>
		<div class="flex items-center gap-2 mb-4">
			<span
				class="inline-block w-1.5 h-1.5 rounded-full"
				style="background-color: {colorVars.color};"
			></span>
			<h2
				class="font-mono text-[11px] uppercase tracking-widest font-medium text-[var(--text-secondary)]"
			>
				Scripts
			</h2>
			<span
				class="font-mono text-[10px] tabular-nums text-[var(--text-muted)] bg-[var(--bg-muted)] rounded-full px-2 py-0.5"
			>
				{data.detail.scripts.length}
			</span>
		</div>

		{#if data.detail.scripts.length === 0}
			<div
				class="text-center py-12 bg-[var(--bg-subtle)] rounded-[var(--radius-md)] border border-dashed border-[var(--border)]"
			>
				<FileCode class="mx-auto text-[var(--text-muted)] mb-3" size={32} strokeWidth={1.75} />
				<p class="font-serif italic text-[var(--text-secondary)]">No scripts found</p>
			</div>
		{:else}
			<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-5">
				{#each data.detail.scripts as script}
					<div
						class="bg-[var(--bg-base)] border border-[var(--border)] rounded-[var(--radius-md)] p-5 hover:border-[var(--border-hover)] transition-colors"
						style="border-left: 3px solid {colorVars.color};"
					>
						<!-- Filename -->
						<div class="flex items-start gap-3 mb-3">
							<FileCode
								size={16}
								strokeWidth={1.75}
								style="color: {colorVars.color};"
								class="flex-shrink-0 mt-0.5"
							/>
							<div class="flex-1 min-w-0">
								<h3
									class="text-sm font-semibold text-[var(--text-primary)] truncate"
									title={script.filename}
								>
									{script.filename}
								</h3>
								{#if script.full_path}
									<p
										class="font-mono text-[10px] text-[var(--text-faint)] truncate mt-0.5"
										title={script.full_path}
									>
										{script.full_path}
									</p>
								{/if}
							</div>
						</div>

						<!-- Language -->
						<div class="mb-3 font-mono text-[10px] uppercase tracking-widest text-[var(--text-muted)]">
							{script.language}
						</div>

						<!-- Event Types -->
						<div class="mb-3">
							<p class="font-mono text-[10px] uppercase tracking-widest text-[var(--text-muted)] mb-2">
								Event types
							</p>
							<div class="flex flex-wrap gap-1">
								{#each script.event_types as eventType}
									<a
										href="/hooks/{eventType}"
										class="font-mono text-[10px] uppercase tracking-widest px-1.5 py-0.5 rounded-[var(--radius-xs)] border border-transparent hover:border-[var(--border)] transition-colors"
										style="background-color: {colorVars.subtle}; color: {colorVars.color};"
									>
										{eventType}
									</a>
								{/each}
							</div>
						</div>

						<!-- Registration Count -->
						<div
							class="flex items-center justify-between pt-3 border-t border-[var(--border)]"
						>
							<span
								class="font-mono text-[10px] uppercase tracking-widest text-[var(--text-muted)]"
								>Registrations</span
							>
							<span
								class="text-sm font-semibold text-[var(--text-primary)] tabular-nums"
							>
								{script.registrations}
							</span>
						</div>

						<!-- Symlink Info -->
						{#if script.is_symlink && script.symlink_target}
							<div
								class="mt-3 pt-3 border-t border-[var(--border)] flex items-start gap-2"
							>
								<LinkIcon
									size={12}
									strokeWidth={1.75}
									class="text-[var(--text-muted)] flex-shrink-0 mt-0.5"
								/>
								<div class="flex-1 min-w-0">
									<p
										class="font-mono text-[10px] uppercase tracking-widest text-[var(--text-muted)]"
									>
										Symlink to
									</p>
									<p
										class="font-mono text-[11px] text-[var(--text-secondary)] truncate mt-0.5"
										title={script.symlink_target}
									>
										{script.symlink_target}
									</p>
								</div>
							</div>
						{/if}
					</div>
				{/each}
			</div>
		{/if}
	</div>
</div>
