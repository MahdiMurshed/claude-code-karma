<script lang="ts">
	import SkeletonBox from './SkeletonBox.svelte';
	import SkeletonText from './SkeletonText.svelte';

	interface Props {
		class?: string;
	}

	let { class: className = '' }: Props = $props();
</script>

<div
	class="
		block
		bg-[var(--bg-base)]
		border border-[var(--border)]
		rounded-[var(--radius-md)]
		px-5 pt-[18px] pb-4
		relative overflow-hidden
		{className}
	"
	role="status"
	aria-label="Loading plugin..."
>
	<!-- Header row: Icon + optional Badge -->
	<div class="flex items-start justify-between mb-[14px]">
		<SkeletonBox width="32px" height="32px" rounded="sm" />
		<SkeletonBox width="56px" height="16px" rounded="sm" />
	</div>

	<!-- Plugin name (serif italic, 22px) -->
	<SkeletonText width="65%" size="lg" class="mb-2" />

	<!-- Description (two-line) -->
	<SkeletonText lines={2} size="xs" class="mb-3" />

	<!-- Metadata eyebrow (counts) -->
	<div class="flex items-center gap-3 py-[10px] border-t border-[var(--border-subtle)]">
		<SkeletonText width="60px" size="xs" />
		<SkeletonText width="60px" size="xs" />
		<SkeletonText width="60px" size="xs" />
	</div>

	<!-- Footer -->
	<div
		class="flex items-center justify-between pt-[10px] border-t border-[var(--border-subtle)]"
	>
		<SkeletonText width="80px" size="xs" />
		<SkeletonText width="90px" size="xs" />
	</div>
</div>
