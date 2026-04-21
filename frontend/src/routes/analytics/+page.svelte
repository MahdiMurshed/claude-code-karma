<script lang="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
	import { Activity, Zap, Database, Cpu, FolderOpen, Clock, BarChart3 } from 'lucide-svelte';
	import { goto } from '$app/navigation';
	import { page, navigating } from '$app/stores';
	import SkeletonBox from '$lib/components/skeleton/SkeletonBox.svelte';
	import SkeletonText from '$lib/components/skeleton/SkeletonText.svelte';
	import SkeletonStatsCard from '$lib/components/skeleton/SkeletonStatsCard.svelte';
	import { getThemeColors } from '$lib/components/charts/chartConfig';
	import TimeFilterDropdown from '$lib/components/TimeFilterDropdown.svelte';
	import PageHeader from '$lib/components/layout/PageHeader.svelte';
	import StatsGrid from '$lib/components/StatsGrid.svelte';
	import CollapsibleGroup from '$lib/components/ui/CollapsibleGroup.svelte';
	import type { AnalyticsFilterPeriod, StatItem } from '$lib/api-types';
	import {
		analyticsFilterOptions,
		getTimestampRangeForFilter,
		isHourBasedFilter,
		getAnalyticsFilterLabel
	} from '$lib/utils';

	// Read filter directly from URL
	let selectedFilter = $derived.by((): AnalyticsFilterPeriod => {
		const filterParam = $page.url.searchParams.get('filter');
		if (filterParam && analyticsFilterOptions.some((o) => o.id === filterParam)) {
			return filterParam as AnalyticsFilterPeriod;
		}
		return 'all';
	});

	const handleFilterChange = (filter: AnalyticsFilterPeriod) => {
		const url = new URL($page.url);
		const range = getTimestampRangeForFilter(filter);

		// Always include timezone offset for accurate local date grouping
		if (browser) {
			url.searchParams.set('tz_offset', new Date().getTimezoneOffset().toString());
		}

		if (filter === 'all') {
			url.searchParams.delete('filter');
			url.searchParams.delete('start_ts');
			url.searchParams.delete('end_ts');
		} else {
			url.searchParams.set('filter', filter);
			if (range) {
				url.searchParams.set('start_ts', range.start.toString());
				url.searchParams.set('end_ts', range.end.toString());
			}
		}

		if (browser) {
			window.location.href = url.toString();
		} else {
			goto(url.toString(), { keepFocus: true });
		}
	};

	interface Analytics {
		total_sessions: number;
		total_tokens: number;
		total_input_tokens: number;
		total_output_tokens: number;
		total_duration_seconds: number;
		estimated_cost_usd: number;
		models_used: Record<string, number>;
		cache_hit_rate: number;
		tools_used: Record<string, number>;
		sessions_by_date: Record<string, number>;
		projects_active: number;
		temporal_heatmap: number[][];
		peak_hours: number[];
		models_categorized: Record<string, number>;
		time_distribution: {
			morning_pct: number;
			afternoon_pct: number;
			evening_pct: number;
			night_pct: number;
			dominant_period: string;
		};
	}

	let { data } = $props();

	// Default analytics object to prevent SSR errors when data is undefined
	const defaultAnalytics: Analytics = {
		total_sessions: 0,
		total_tokens: 0,
		total_input_tokens: 0,
		total_output_tokens: 0,
		total_duration_seconds: 0,
		estimated_cost_usd: 0,
		models_used: {},
		cache_hit_rate: 0,
		tools_used: {},
		sessions_by_date: {},
		projects_active: 0,
		temporal_heatmap: [],
		peak_hours: [],
		models_categorized: {},
		time_distribution: {
			morning_pct: 0,
			afternoon_pct: 0,
			evening_pct: 0,
			night_pct: 0,
			dominant_period: ''
		}
	};

	// Merge actual data with defaults to ensure all properties exist
	let analytics = $derived.by(() => {
		const rawAnalytics = data.analytics as unknown as Analytics | undefined;
		if (!rawAnalytics) return defaultAnalytics;

		return {
			...defaultAnalytics,
			...rawAnalytics,
			time_distribution: {
				...defaultAnalytics.time_distribution,
				...(rawAnalytics.time_distribution ?? {})
			}
		};
	});

	// --- Helpers ---
	const formatK = (n: number) => {
		if (n >= 1000000) return (n / 1000000).toFixed(1) + 'M';
		if (n >= 1000) return (n / 1000).toFixed(1) + 'K';
		return n.toString();
	};

	const formatCurrency = (n: number) =>
		new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(n);

	// --- Date Processing ---
	let sortedDates = $derived(Object.keys(analytics.sessions_by_date).sort());

	// Stats - context-aware based on filter
	let last30Days = $derived(sortedDates.slice(-30));

	// For hour-based filters, show sessions per hour; for day-based, show per day
	let avgDisplay = $derived.by(() => {
		const numDays = sortedDates.length;
		const totalSessions = analytics.total_sessions;

		if (numDays === 0 || totalSessions === 0) {
			return { value: '0', unit: '/day' };
		}

		// For single-day (hour-based) filters, calculate per-hour rate
		if (isHourBasedFilter(selectedFilter)) {
			const hours = parseInt(selectedFilter) || 6;
			const perHour = (totalSessions / hours).toFixed(1);
			return { value: perHour, unit: '/hour' };
		}

		// For multi-day filters, show per-day average
		const perDay = (totalSessions / numDays).toFixed(1);
		return { value: perDay, unit: '/day' };
	});

	// Legacy movingAvg for backward compatibility
	let movingAvg = $derived(
		last30Days.length > 0
			? (
					last30Days.reduce((sum, d) => sum + (analytics.sessions_by_date[d] || 0), 0) /
					last30Days.length
				).toFixed(1)
			: '0'
	);

	let tokensPerSession = $derived(
		analytics.total_sessions > 0
			? Math.round(analytics.total_tokens / analytics.total_sessions)
			: 0
	);

	// Sparkline
	let sparklineData = $derived(
		sortedDates.slice(-14).map((d) => analytics.sessions_by_date[d] || 0)
	);
	let sparkMax = $derived(Math.max(...sparklineData, 1));

	// --- Model Distribution ---
	// Use models_categorized if it has data, otherwise fall back to models_used
	let modelsData = $derived(() => {
		const categorized = analytics.models_categorized;
		const used = analytics.models_used;
		// Check for non-empty objects (empty {} is truthy but has no keys)
		if (categorized && Object.keys(categorized).length > 0) return categorized;
		if (used && Object.keys(used).length > 0) return used;
		return {};
	});

	let modelDist = $derived(
		Object.entries(modelsData())
			.filter(([_, count]) => count > 0)
			.sort((a, b) => b[1] - a[1])
			.map(([name, count]) => {
				const total = Object.values(modelsData()).reduce((a, b) => a + b, 0);
				return { name, count, perc: total > 0 ? (count / total) * 100 : 0 };
			})
			.filter((m) => m.perc >= 2)
	); // Hide <2%

	const getModelColor = (name: string) => {
		const lower = name.toLowerCase();
		if (lower.includes('opus')) return 'var(--model-opus)';
		if (lower.includes('sonnet')) return 'var(--model-sonnet)';
		if (lower.includes('haiku')) return 'var(--model-haiku)';
		return 'var(--nav-teal)'; // Teal ink for 'Other' models
	};

	// --- Cache ---
	let cacheHitPercent = $derived((analytics.cache_hit_rate * 100).toFixed(1));

	// --- Cost ---
	let costPerSession = $derived(
		analytics.total_sessions ? analytics.estimated_cost_usd / analytics.total_sessions : 0
	);

	// Format peak hours array (e.g., [9, 10, 11]) to readable range "9am-12pm"
	const formatPeakHours = (hours: number[]) => {
		if (!hours || hours.length === 0) return '—';
		const sorted = [...hours].sort((a, b) => a - b);
		const start = sorted[0];
		const end = sorted[sorted.length - 1] + 1; // +1 because it's the *end* of the hour
		const formatHour = (h: number) => {
			const hour12 = h % 12 || 12;
			const ampm = h < 12 ? 'am' : 'pm';
			return `${hour12}${ampm}`;
		};
		return `${formatHour(start)}–${formatHour(end % 24)}`;
	};

	// Calculate absolute hours from percentage based on total duration
	const formatHoursFromPct = (pct: number) => {
		const totalHours = analytics.total_duration_seconds / 3600;
		const hours = (pct / 100) * totalHours;
		return hours >= 1 ? `${hours.toFixed(0)}h` : `${(hours * 60).toFixed(0)}m`;
	};

	// Format date string (YYYY-MM-DD) without timezone shift
	// Using T12:00:00 to avoid day boundary issues in any timezone
	const formatDateLabel = (dateStr: string): string => {
		const d = new Date(dateStr + 'T12:00:00');
		return d.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
	};

	// --- Hero Stats ---
	let stats = $derived<StatItem[]>([
		{
			title: 'Total Sessions',
			value: analytics.total_sessions.toLocaleString(),
			icon: Activity,
			color: 'purple'
		},
		{
			title: 'Total Cost',
			value: formatCurrency(analytics.estimated_cost_usd),
			icon: Zap,
			color: 'green'
		},
		{
			title: 'Cache Hit Rate',
			value: `${cacheHitPercent}%`,
			icon: Database,
			color: 'blue'
		}
	]);

	let isPageLoading = $derived(!!$navigating && $navigating.to?.route.id === '/analytics');

	// --- Collapsible Group State ---
	const groupKeys = ['velocity', 'efficiency', 'rhythm'] as const;
	let expandedGroups = $state<Set<string>>(new Set(groupKeys));

	function toggleGroup(key: string) {
		if (expandedGroups.has(key)) {
			expandedGroups.delete(key);
		} else {
			expandedGroups.add(key);
		}
		expandedGroups = new Set(expandedGroups);
	}

	// Chart
	// svelte-ignore non_reactive_update: chartCanvas is only bound once during mount
	let chartCanvas: HTMLCanvasElement;
	let chartInstance = $state<any>(null);

	$effect(() => {
		if (chartInstance && sortedDates) {
			const newCounts = sortedDates.map((date) => analytics.sessions_by_date[date]);
			const newLabels = sortedDates.map((date) => formatDateLabel(date));

			chartInstance.data.labels = newLabels;
			chartInstance.data.datasets[0].data = newCounts;
			chartInstance.update();
		}
	});

	onMount(async () => {
		const Chart = (await import('chart.js/auto')).default;
		const c = getThemeColors();

		const sessionCounts = sortedDates.map((date) => analytics.sessions_by_date[date]);

		chartInstance = new Chart(chartCanvas, {
			type: 'bar',
			data: {
				labels: sortedDates.map((date) => formatDateLabel(date)),
				datasets: [
					{
						label: 'Sessions',
						data: sessionCounts,
						backgroundColor: c.primary,
						hoverBackgroundColor: c.text,
						borderRadius: 2,
						barThickness: 'flex',
						maxBarThickness: 14
					}
				]
			},
			options: {
				responsive: true,
				maintainAspectRatio: false,
				plugins: {
					legend: { display: false },
					tooltip: {
						backgroundColor: c.bgBase,
						titleColor: c.text,
						bodyColor: c.textSecondary,
						borderColor: c.border,
						borderWidth: 1,
						padding: 10,
						cornerRadius: 4,
						displayColors: false,
						titleFont: { family: "'Geist', system-ui, sans-serif", weight: 600, size: 12 },
						bodyFont: { family: "'Geist Mono', ui-monospace, monospace", size: 11 }
					}
				},
				scales: {
					y: { beginAtZero: true, grid: { color: c.border }, ticks: { display: false } },
					x: {
						grid: { display: false },
						ticks: {
							font: { family: "'Geist Mono', ui-monospace, monospace", size: 10 },
							color: c.textMuted,
							maxTicksLimit: 8,
							maxRotation: 0
						}
					}
				}
			}
		});
	});
</script>

<div class="max-w-[1100px] mx-auto space-y-6">
	{#if isPageLoading}
		<div class="space-y-6" role="status" aria-busy="true" aria-label="Loading...">
			<!-- Page Header skeleton -->
			<div class="flex items-start justify-between">
				<div>
					<div class="flex items-center gap-2 mb-2">
						<SkeletonText width="70px" size="xs" />
						<span class="text-[var(--text-muted)]">/</span>
						<SkeletonText width="80px" size="xs" />
					</div>
					<div class="flex items-center gap-4">
						<SkeletonBox width="48px" height="48px" rounded="lg" />
						<div>
							<SkeletonText width="120px" size="xl" class="mb-2" />
							<SkeletonText width="260px" size="sm" />
						</div>
					</div>
				</div>
				<SkeletonBox width="140px" height="36px" rounded="md" />
			</div>

			<!-- Hero Stats skeleton (3 cols) -->
			<div class="grid grid-cols-1 sm:grid-cols-3 gap-4">
				{#each Array(3) as _}
					<SkeletonStatsCard />
				{/each}
			</div>

			<!-- Velocity group skeleton -->
			<div class="border border-[var(--border)] rounded-[var(--radius-lg)] overflow-hidden bg-[var(--bg-base)]">
				<div class="flex items-center gap-3 px-4 py-4">
					<SkeletonBox width="32px" height="32px" rounded="md" />
					<SkeletonText width="100px" size="sm" />
					<div class="flex-1"></div>
					<SkeletonText width="80px" size="xs" />
				</div>
				<div class="border-t border-[var(--border)] p-4">
					<SkeletonBox height="160px" rounded="lg" />
				</div>
			</div>

			<!-- Efficiency group skeleton -->
			<div class="border border-[var(--border)] rounded-[var(--radius-lg)] overflow-hidden bg-[var(--bg-base)]">
				<div class="flex items-center gap-3 px-4 py-4">
					<SkeletonBox width="32px" height="32px" rounded="md" />
					<SkeletonText width="100px" size="sm" />
					<div class="flex-1"></div>
					<SkeletonText width="80px" size="xs" />
				</div>
				<div class="border-t border-[var(--border)] p-4">
					<div class="grid grid-cols-1 lg:grid-cols-4 gap-4">
						{#each Array(4) as _}
							<SkeletonBox height="120px" rounded="lg" />
						{/each}
					</div>
				</div>
			</div>

			<!-- Rhythm group skeleton -->
			<div class="border border-[var(--border)] rounded-[var(--radius-lg)] overflow-hidden bg-[var(--bg-base)]">
				<div class="flex items-center gap-3 px-4 py-4">
					<SkeletonBox width="32px" height="32px" rounded="md" />
					<SkeletonText width="80px" size="sm" />
					<div class="flex-1"></div>
					<SkeletonText width="80px" size="xs" />
				</div>
				<div class="border-t border-[var(--border)] p-4 space-y-3">
					{#each Array(4) as _}
						<div class="flex items-center gap-3">
							<SkeletonText width="80px" size="xs" />
							<SkeletonBox height="6px" rounded="full" class="flex-1" />
							<SkeletonText width="80px" size="xs" />
						</div>
					{/each}
				</div>
			</div>
		</div>
	{:else}
	<!-- Page Header with Breadcrumb -->
	<PageHeader
		title="Analytics"
		icon={BarChart3}
		iconColor="--nav-green"
		breadcrumbs={[{ label: 'Dashboard', href: '/' }, { label: 'Analytics' }]}
		subtitle="Your coding patterns and AI collaboration"
	>
		{#snippet headerRight()}
			<TimeFilterDropdown {selectedFilter} onFilterChange={handleFilterChange} />
		{/snippet}
	</PageHeader>

	{#if analytics.total_sessions === 0}
		<!-- Editorial empty state — stands alone, no zero-value cards -->
		<div
			class="border border-[var(--border)] rounded-[var(--radius-lg)] bg-[var(--bg-base)] px-6 py-20 text-center"
		>
			<p
				class="text-[var(--text-secondary)] text-[26px] leading-snug"
				style="font-family: var(--font-serif); font-style: italic;"
			>
				No sessions in this range.
			</p>
			<p
				class="mt-4 text-[10px] font-mono uppercase tracking-[0.16em] text-[var(--text-muted)]"
			>
				Try a wider window
			</p>
		</div>
	{:else}

	<!-- Hero Stats Row -->
	<StatsGrid {stats} columns={3} />

	<!-- Group 1: Velocity — Activity bar chart -->
	<CollapsibleGroup
		title="Velocity"
		open={expandedGroups.has('velocity')}
		onOpenChange={() => toggleGroup('velocity')}
	>
		{#snippet icon()}
			<div class="p-1.5 bg-[var(--bg-subtle)] rounded-md">
				<Activity size={14} class="text-[var(--text-muted)]" />
			</div>
		{/snippet}
		{#snippet metadata()}
			<div class="flex items-center gap-3">
				<span class="eyebrow-thin">
					Avg
					<span class="font-mono tabular-nums text-[var(--text-secondary)] ml-1"
						>{avgDisplay.value}</span
					><span class="text-[var(--text-faint)]">{avgDisplay.unit}</span>
				</span>
				<span class="hidden sm:flex items-end gap-[2px] h-5" aria-hidden="true">
					{#each sparklineData as val, i}
						<span
							class="inline-block w-[2px]"
							style="height: {Math.max(
								2,
								(val / sparkMax) * 100
							)}%; background-color: var(--accent); opacity: {0.3 +
								(i / sparklineData.length) * 0.7};"
						></span>
					{/each}
				</span>
			</div>
		{/snippet}

		<div class="space-y-4">
			<!-- Token context row — mono-caps eyebrows with 6px dot markers -->
			<div class="flex items-center gap-4 text-[11px]">
				<span class="eyebrow flex items-center gap-2">
					<span class="dot-6" style="background-color: var(--accent);"></span>
					<span>Tokens</span>
					<span class="font-mono tabular-nums text-[var(--text-primary)] ml-0.5 normal-case tracking-normal"
						>{formatK(analytics.total_tokens)}</span
					>
				</span>
				<span class="eyebrow flex items-center gap-2">
					<span class="dot-6" style="background-color: var(--nav-green);"></span>
					<span>Hours</span>
					<span class="font-mono tabular-nums text-[var(--text-primary)] ml-0.5 normal-case tracking-normal"
						>{(analytics.total_duration_seconds / 3600).toFixed(0)}h</span
					>
				</span>
				<span class="eyebrow flex items-center gap-2">
					<span class="dot-6" style="background-color: var(--nav-blue);"></span>
					<span>Per Sess</span>
					<span class="font-mono tabular-nums text-[var(--text-primary)] ml-0.5 normal-case tracking-normal"
						>{formatK(tokensPerSession)}</span
					>
				</span>
			</div>

			<!-- Bar chart -->
			<div class="h-40 w-full">
				<canvas bind:this={chartCanvas}></canvas>
			</div>
		</div>
	</CollapsibleGroup>

	<!-- Group 2: Efficiency — Cache, Projects, Compute DNA -->
	<CollapsibleGroup
		title="Efficiency"
		open={expandedGroups.has('efficiency')}
		onOpenChange={() => toggleGroup('efficiency')}
	>
		{#snippet icon()}
			<div class="p-1.5 bg-[var(--bg-subtle)] rounded-md">
				<Zap size={14} class="text-[var(--text-muted)]" />
			</div>
		{/snippet}
		{#snippet metadata()}
			<span class="text-xs text-[var(--text-muted)] tabular-nums">
				{getAnalyticsFilterLabel(selectedFilter)}
			</span>
		{/snippet}

		<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-3">
			<!-- Cache Card -->
			<div class="p-4 bg-[var(--bg-base)] border border-[var(--border)] rounded-[var(--radius-lg)]">
				<div class="flex items-center justify-between mb-3">
					<div class="flex items-center gap-1.5 eyebrow">
						<Database size={12} class="text-[var(--text-muted)]" />
						<span>Cache</span>
					</div>
					{#if analytics.cache_hit_rate > 0.85}
						<span
							class="px-1.5 py-0.5 bg-[var(--success-subtle)] rounded text-[9px] font-medium uppercase tracking-[0.14em] text-[var(--success)]"
						>
							Excellent
						</span>
					{/if}
				</div>
				<div class="flex items-baseline gap-1.5 mb-3">
					<span class="value-lg">{cacheHitPercent}%</span>
					<span class="text-[11px] text-[var(--text-muted)]">hit rate</span>
				</div>
				<div
					class="relative w-full h-[3px] bg-[var(--bg-subtle)] rounded-full overflow-hidden"
				>
					<div
						class="absolute top-0 left-0 h-full rounded-full bg-[var(--accent)]"
						style="width: {analytics.cache_hit_rate * 100}%"
					></div>
				</div>
			</div>

			<!-- Cost Card -->
			<div class="p-4 bg-[var(--bg-base)] border border-[var(--border)] rounded-[var(--radius-lg)]">
				<div class="flex items-center gap-1.5 eyebrow mb-3">
					<Zap size={12} class="text-[var(--text-muted)]" />
					<span>Cost / Session</span>
				</div>
				<div class="flex items-baseline gap-1">
					<span class="text-[11px] font-mono text-[var(--text-muted)]">$</span>
					<span class="value-lg">{costPerSession.toFixed(2)}</span>
				</div>
				<div class="mt-2 text-[11px] text-[var(--text-muted)]">
					<span class="font-mono tabular-nums text-[var(--text-secondary)]"
						>{formatCurrency(analytics.estimated_cost_usd)}</span
					>
					total
				</div>
			</div>

			<!-- Projects Card -->
			<div class="p-4 bg-[var(--bg-base)] border border-[var(--border)] rounded-[var(--radius-lg)]">
				<div class="flex items-center gap-1.5 eyebrow mb-3">
					<FolderOpen size={12} class="text-[var(--text-muted)]" />
					<span>Projects</span>
				</div>
				<div class="flex items-baseline gap-1.5">
					<span class="value-lg">{analytics.projects_active}</span>
					<span class="text-[11px] text-[var(--text-muted)]">worked on</span>
				</div>
			</div>

			<!-- Compute DNA Card -->
			<div
				class="p-4 bg-[var(--bg-base)] border border-[var(--border)] rounded-[var(--radius-lg)]"
			>
				<div class="flex items-center gap-1.5 eyebrow mb-3">
					<Cpu size={12} class="text-[var(--text-muted)]" />
					<span>Compute DNA</span>
				</div>
				{#if modelDist.length > 0}
					<div class="w-full flex h-[10px] rounded overflow-hidden mb-3 bg-[var(--bg-subtle)]">
						{#each modelDist as model}
							<div
								class="h-full"
								style="width: {model.perc}%; background-color: {getModelColor(
									model.name
								)};"
								title="{model.name}: {model.perc.toFixed(0)}%"
							></div>
						{/each}
					</div>
					<div class="flex flex-col gap-1 text-[11px]">
						{#each modelDist as model}
							<div class="flex items-center gap-1.5">
								<span
									class="dot-6"
									style="background-color: {getModelColor(model.name)}"
								></span>
								<span class="text-[var(--text-secondary)] truncate">{model.name}</span>
								<span class="ml-auto font-mono tabular-nums text-[var(--text-muted)]"
									>{model.perc.toFixed(0)}%</span
								>
							</div>
						{/each}
					</div>
				{:else}
					<p class="text-[11px] italic text-[var(--text-faint)]" style="font-family: var(--font-serif);">No model data</p>
				{/if}
			</div>
		</div>
	</CollapsibleGroup>

	<!-- Group 3: Rhythm — Time Distribution -->
	<CollapsibleGroup
		title="Rhythm"
		open={expandedGroups.has('rhythm')}
		onOpenChange={() => toggleGroup('rhythm')}
	>
		{#snippet icon()}
			<div class="p-1.5 bg-[var(--bg-subtle)] rounded-md">
				<Clock size={14} class="text-[var(--text-muted)]" />
			</div>
		{/snippet}
		{#snippet metadata()}
			<div class="flex items-center gap-2 text-[11px] text-[var(--text-muted)]">
				<span class="eyebrow-thin">Peak</span>
				<span class="peak-callout">{formatPeakHours(analytics.peak_hours)}</span>
			</div>
		{/snippet}

		<div class="flex flex-col gap-5">
			{#each [
				{ label: 'Morning', range: '06:00–12:00', pct: analytics.time_distribution.morning_pct },
				{ label: 'Afternoon', range: '12:00–18:00', pct: analytics.time_distribution.afternoon_pct },
				{ label: 'Evening', range: '18:00–24:00', pct: analytics.time_distribution.evening_pct },
				{ label: 'Night', range: '00:00–06:00', pct: analytics.time_distribution.night_pct }
			] as period}
				<div>
					<div class="flex items-baseline justify-between mb-1.5">
						<div class="eyebrow flex items-center gap-2">
							<span class="dot-6" style="background-color: var(--accent);"></span>
							<span>{period.label}</span>
							<span class="text-[var(--text-faint)]">·</span>
							<span class="tracking-normal normal-case font-mono text-[var(--text-faint)]"
								>{period.range}</span
							>
						</div>
						<span class="font-mono tabular-nums text-[11px] text-[var(--text-secondary)]"
							>{period.pct.toFixed(0)}%
							<span class="text-[var(--text-faint)]"
								>· {formatHoursFromPct(period.pct)}</span
							></span
						>
					</div>
					<div
						class="relative w-full h-[3px] bg-[var(--bg-subtle)] rounded-full overflow-hidden"
					>
						<div
							class="absolute top-0 left-0 h-full bg-[var(--accent)] rounded-full"
							style="width: {period.pct}%"
						></div>
					</div>
				</div>
			{/each}
		</div>

		<!-- Footer: hairline rule + total -->
		<div
			class="mt-5 pt-3 border-t border-[var(--border)] flex items-center justify-between text-[11px] text-[var(--text-muted)]"
		>
			<span class="eyebrow-thin">Total</span>
			<span class="font-mono tabular-nums text-[var(--text-secondary)]"
				>{(analytics.total_duration_seconds / 3600).toFixed(0)}h</span
			>
		</div>
	</CollapsibleGroup>
	{/if}
	{/if}
</div>

<style>
	.eyebrow {
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.16em;
		text-transform: uppercase;
		font-weight: 500;
		color: var(--text-muted);
	}

	.eyebrow-thin {
		font-family: var(--font-mono);
		font-size: 10px;
		letter-spacing: 0.14em;
		text-transform: uppercase;
		color: var(--text-muted);
	}

	.value-lg {
		font-family: var(--font-mono);
		font-size: 22px;
		font-weight: 600;
		letter-spacing: -0.01em;
		color: var(--text-primary);
		font-variant-numeric: tabular-nums;
		line-height: 1.1;
	}

	.dot-6 {
		display: inline-block;
		width: 6px;
		height: 6px;
		border-radius: 50%;
		flex-shrink: 0;
	}

	.peak-callout {
		font-family: var(--font-serif);
		font-style: italic;
		font-size: 15px;
		color: var(--text-primary);
		letter-spacing: -0.005em;
		line-height: 1;
	}

	/* Tabular-nums default for this page */
	:global(.tabular-nums),
	.tabular-nums {
		font-variant-numeric: tabular-nums;
	}
</style>
