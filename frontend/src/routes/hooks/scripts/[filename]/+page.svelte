<script lang="ts">
	import DOMPurify from 'isomorphic-dompurify';
	import {
		FileCode,
		Copy,
		Check,
		ExternalLink,
		FolderSymlink,
		Hash,
		AlertTriangle
	} from 'lucide-svelte';
	import PageHeader from '$lib/components/layout/PageHeader.svelte';
	import Badge from '$lib/components/ui/Badge.svelte';
	import EmptyState from '$lib/components/ui/EmptyState.svelte';
	import { getHookSourceColorVars } from '$lib/utils';

	let { data } = $props();

	let detail = $derived(data.detail);
	let script = $derived(detail.script);
	let sourceColors = $derived(getHookSourceColorVars(detail.source_type, script.source_name));

	// Language display names
	const languageLabels: Record<string, string> = {
		python: 'Python',
		node: 'Node.js',
		shell: 'Shell',
		bash: 'Shell'
	};

	let languageLabel = $derived(languageLabels[script.language] || script.language);

	function formatBytes(bytes: number): string {
		if (bytes < 1024) return `${bytes} B`;
		if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
		return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
	}

	// Copy button state
	let copied = $state(false);

	async function copyCode() {
		if (!detail.content) return;
		try {
			await navigator.clipboard.writeText(detail.content);
			copied = true;
			setTimeout(() => (copied = false), 2000);
		} catch {
			// Clipboard API not available
		}
	}

	// Format modified date
	let modifiedLabel = $derived(
		detail.modified_at
			? new Date(detail.modified_at).toLocaleDateString('en-US', {
					year: 'numeric',
					month: 'short',
					day: 'numeric',
					hour: '2-digit',
					minute: '2-digit'
				})
			: null
	);
</script>

<svelte:head>
	<title>{data.filename} - Hook Script - Claude Code Karma</title>
</svelte:head>

<div class="max-w-5xl mx-auto">
	<PageHeader
		title={script.filename}
		icon={FileCode}
		iconColorRaw={sourceColors}
		breadcrumbs={[
			{ label: 'Dashboard', href: '/' },
			{ label: 'Hooks', href: '/hooks' },
			{ label: script.filename }
		]}
		subtitle={script.full_path || undefined}
		metadata={[
			...(script.is_symlink && script.symlink_target
				? [{ icon: FolderSymlink, text: script.symlink_target }]
				: []),
			...(detail.line_count ? [{ icon: Hash, text: `${detail.line_count} lines` }] : []),
			...(detail.size_bytes ? [{ text: formatBytes(detail.size_bytes) }] : []),
			...(modifiedLabel ? [{ text: `Modified ${modifiedLabel}` }] : [])
		]}
	>
		{#snippet badges()}
			<Badge variant="slate">{languageLabel}</Badge>
			<Badge variant="slate">{detail.source_type}</Badge>
			<Badge variant="slate"
				>{script.registrations} registration{script.registrations !== 1 ? 's' : ''}</Badge
			>
		{/snippet}
	</PageHeader>

	<!-- Event Types -->
	{#if script.event_types.length > 0}
		<section class="mb-8">
			<div class="flex items-center gap-2 mb-3">
				<span
					class="inline-block w-1.5 h-1.5 rounded-full"
					style="background-color: var(--nav-orange);"
				></span>
				<h2
					class="font-mono text-[11px] uppercase tracking-widest font-medium text-[var(--text-secondary)]"
				>
					Event types
				</h2>
			</div>
			<div class="flex flex-wrap gap-1.5">
				{#each script.event_types as eventType}
					<a
						href="/hooks/{encodeURIComponent(eventType)}"
						class="
							inline-flex items-center gap-1.5
							px-2 py-1
							font-mono text-[10px] uppercase tracking-widest
							rounded-[var(--radius-xs)]
							bg-[var(--bg-subtle)] text-[var(--text-secondary)]
							border border-[var(--border)]
							hover:bg-[var(--nav-orange-subtle)] hover:text-[var(--nav-orange)] hover:border-[var(--nav-orange)]
							transition-colors
						"
					>
						{eventType}
						<ExternalLink size={10} strokeWidth={1.75} />
					</a>
				{/each}
			</div>
		</section>
	{/if}

	<!-- Source Code -->
	<section>
		<div
			class="
				border border-[var(--border)]
				rounded-[var(--radius-md)]
				overflow-hidden
				bg-[var(--bg-base)]
			"
		>
			<!-- Code Header -->
			<div
				class="
					flex items-center justify-between
					px-4 py-2.5
					border-b border-[var(--border)]
					bg-[var(--bg-subtle)]
				"
			>
				<div
					class="flex items-center gap-2 font-mono text-[10px] uppercase tracking-widest text-[var(--text-muted)]"
				>
					<FileCode size={12} strokeWidth={1.75} />
					<span>{script.filename}</span>
				</div>
				{#if detail.content}
					<button
						onclick={copyCode}
						class="
							flex items-center gap-1.5
							px-2 py-1
							font-mono text-[10px] uppercase tracking-widest
							rounded-[var(--radius-xs)]
							text-[var(--text-muted)]
							hover:text-[var(--accent)]
							hover:bg-[var(--bg-base)]
							border border-transparent
							hover:border-[var(--border)]
							transition-colors
						"
					>
						{#if copied}
							<Check size={12} strokeWidth={1.75} style="color: var(--success);" />
							<span style="color: var(--success);">Copied</span>
						{:else}
							<Copy size={12} strokeWidth={1.75} />
							<span>Copy</span>
						{/if}
					</button>
				{/if}
			</div>

			<!-- Code Content -->
			{#if detail.error === 'file_not_found'}
				<div class="p-8">
					<EmptyState
						icon={AlertTriangle}
						title="Script file not found"
						description="The file at {script.full_path ||
							'unknown path'} could not be found. It may have been moved or deleted."
					/>
				</div>
			{:else if detail.error === 'file_too_large'}
				<div class="p-8">
					<EmptyState
						icon={AlertTriangle}
						title="File too large to display"
						description="This script exceeds the 500KB display limit ({detail.size_bytes
							? formatBytes(detail.size_bytes)
							: 'unknown size'})."
					/>
				</div>
			{:else if detail.error === 'binary_file'}
				<div class="p-8">
					<EmptyState
						icon={AlertTriangle}
						title="Binary file"
						description="This file contains binary content that cannot be displayed as text."
					/>
				</div>
			{:else if data.highlightedHtml}
				<div class="shiki-container overflow-x-auto">
					{@html DOMPurify.sanitize(data.highlightedHtml)}
				</div>
			{:else if detail.content}
				<pre
					class="p-4 overflow-x-auto text-sm font-mono text-[var(--text-secondary)] leading-relaxed bg-[var(--bg-base)]">{detail.content}</pre>
			{:else}
				<div class="p-8">
					<EmptyState
						icon={FileCode}
						title="No content available"
						description="Unable to load the script content."
					/>
				</div>
			{/if}
		</div>
	</section>
</div>

<style>
	.shiki-container :global(pre) {
		margin: 0;
		padding: 1rem 1.25rem;
		overflow-x: auto;
		font-family: var(--font-mono);
		font-size: 0.8125rem;
		line-height: 1.7;
		tab-size: 4;
	}

	.shiki-container :global(pre code) {
		font-family: inherit;
	}

	.shiki-container :global(.line) {
		display: inline-block;
		width: 100%;
	}
</style>
