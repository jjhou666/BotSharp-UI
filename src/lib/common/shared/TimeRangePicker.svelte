<script>
	import { onMount, createEventDispatcher } from 'svelte';
	import { Button, Input } from '@sveltestrap/sveltestrap';
	import { TIME_RANGE_OPTIONS, CUSTOM_DATE_RANGE, DEFAULT_START_TIME, DEFAULT_END_TIME } from '$lib/helpers/constants';
	import { clickoutsideDirective } from '$lib/helpers/directives';
	import BotsharpTooltip from '$lib/common/tooltip/BotsharpTooltip.svelte';

	// Constants
	const RECENT_TIME_RANGES_KEY = 'botsharp_recent_time_ranges';
	const MAX_RECENT_ITEMS = 10;
	const TAB_RELATIVE = 'relative';
	const TAB_RECENT = 'recent';
	const TAB_CUSTOM = 'custom';
	const TOOLTIP_ID = 'time-range-picker-tooltip';
	const DATE_SEPARATOR = '-';
	const DISPLAY_DATE_SEPARATOR = '/';
	const PAD_LENGTH = 2;
	const PAD_CHAR = '0';

	/** @type {string} */
	export let timeRange = '';

	/** @type {string} */
	export let startDate = '';

	/** @type {string} */
	export let endDate = '';

	/** @type {string} */
	export let startTime = DEFAULT_START_TIME;

	/** @type {string} */
	export let endTime = DEFAULT_END_TIME;

	/** @type {string} */
	let timeRangeDisplayText = '';

	/** @type {boolean} */
	let showDatePicker = false;

	/** @type {string} */
	let datePickerTab = TAB_RELATIVE;

	/** @type {string | null} */
	let validationError = null;

	/** @type {Array<{ startDate: string, endDate: string, startTime: string, endTime: string, label: string, timeRange?: string }>} */
	let recentTimeRanges = [];

	// Preset time range options (exclude custom date range from dropdown)
	const presetTimeRangeOptions = TIME_RANGE_OPTIONS
		.filter(x => x.value !== CUSTOM_DATE_RANGE)
		.map(x => ({
			label: x.label,
			value: x.value
		}));

	onMount(() => {
		loadRecentTimeRanges();
	});

	// Get today's date in YYYY-MM-DD format
	const getTodayStr = () => {
		const d = new Date();
		console.log(d.getFullYear() + DATE_SEPARATOR + String(d.getMonth() + 1).padStart(PAD_LENGTH, PAD_CHAR) + DATE_SEPARATOR + String(d.getDate()).padStart(PAD_LENGTH, PAD_CHAR));
		return d.getFullYear() + DATE_SEPARATOR + String(d.getMonth() + 1).padStart(PAD_LENGTH, PAD_CHAR) + DATE_SEPARATOR + String(d.getDate()).padStart(PAD_LENGTH, PAD_CHAR);
	};

	// Get yesterday's date in YYYY-MM-DD format
	const getYesterdayStr = () => {
		const d = new Date();
		d.setDate(d.getDate() - 1);
		return d.getFullYear() + DATE_SEPARATOR + String(d.getMonth() + 1).padStart(PAD_LENGTH, PAD_CHAR) + DATE_SEPARATOR + String(d.getDate()).padStart(PAD_LENGTH, PAD_CHAR);
	};

	// Format date for display (YYYY-MM-DD -> MM/DD/YYYY)
	const formatDateForDisplay = (/** @type {string} */ dateStr) => {
		if (!dateStr) return '';
		const [year, month, day] = dateStr.split(DATE_SEPARATOR);
		return `${month}${DISPLAY_DATE_SEPARATOR}${day}${DISPLAY_DATE_SEPARATOR}${year}`;
	};

	// Convert 24-hour time to 12-hour format with AM/PM
	const formatTimeTo12Hour = (/** @type {string} */ time24) => {
		if (!time24) return '';
		const [hours, minutes] = time24.split(':').map(Number);
		const period = hours >= 12 ? 'PM' : 'AM';
		const hours12 = hours === 0 ? 12 : hours > 12 ? hours - 12 : hours;
		return `${hours12}:${String(minutes).padStart(2, '0')} ${period}`;
	};

	// Format time range for display (12-hour format with AM/PM)
	const formatTimeRangeForDisplay = (/** @type {string} */ sDate, /** @type {string} */ eDate, /** @type {string} */ sTime, /** @type {string} */ eTime) => {
		const start = formatDateForDisplay(sDate);
		const end = formatDateForDisplay(eDate);

		const startTimeStr = formatTimeTo12Hour(sTime || DEFAULT_START_TIME);
		const endTimeStr = formatTimeTo12Hour(eTime || DEFAULT_END_TIME);

		if (start === end) {
			return `${start} ${startTimeStr} - ${endTimeStr}`;
		} else {
			return `${start} ${startTimeStr} - ${end} ${endTimeStr}`;
		}
	};

	// Format time range for Recent tab display
	const formatRecentTimeRangeLabel = (/** @type {string} */ sDate, /** @type {string} */ eDate, /** @type {string} */ sTime, /** @type {string} */ eTime) => {
		const start = formatDateForDisplay(sDate);
		const end = formatDateForDisplay(eDate);
		const startTimeStr = formatTimeTo12Hour(sTime || DEFAULT_START_TIME);
		const endTimeStr = formatTimeTo12Hour(eTime || DEFAULT_END_TIME);

		if (start === end) {
			return `${start} ${startTimeStr} - ${endTimeStr}`;
		} else {
			return `${start} ${startTimeStr} - ${end} ${endTimeStr}`;
		}
	};

	// Update time range display text reactively
	$: {
		if (timeRange === CUSTOM_DATE_RANGE && startDate && endDate) {
			timeRangeDisplayText = formatTimeRangeForDisplay(startDate, endDate, startTime, endTime);
		} else {
			const selected = presetTimeRangeOptions.find(x => x.value === timeRange);
			timeRangeDisplayText = selected ? selected.label : '';
		}
	}

	// Validate custom date/time selection
	$: {
		validationError = null;
		if (datePickerTab === TAB_CUSTOM && startDate && endDate) {
			const startDateTime = new Date(`${startDate}T${startTime || DEFAULT_START_TIME}`);
			const endDateTime = new Date(`${endDate}T${endTime || DEFAULT_END_TIME}`);
			if (endDateTime < startDateTime) {
				validationError = 'End date/time must be after start date/time';
			}
		}
	}

	const dispatch = createEventDispatcher();

	function loadRecentTimeRanges() {
		try {
			const stored = localStorage.getItem(RECENT_TIME_RANGES_KEY);
			if (stored) {
				recentTimeRanges = JSON.parse(stored);
			}
		} catch (e) {
			recentTimeRanges = [];
		}
	}

	/**
	 * @param {{ startDate: string, endDate: string, startTime: string, endTime: string, timeRange?: string, label?: string }} range
	 */
	function saveRecentTimeRange(range) {
		try {
			// Use provided label or format from dates
			const label = range.label || formatRecentTimeRangeLabel(range.startDate, range.endDate, range.startTime, range.endTime);
			const newRange = { ...range, label };

			// Remove duplicate if exists (check by timeRange if it's a relative range, otherwise by dates)
			if (range.timeRange) {
				recentTimeRanges = recentTimeRanges.filter(r => r.timeRange !== range.timeRange);
			} else {
				recentTimeRanges = recentTimeRanges.filter(r =>
					r.startDate !== range.startDate ||
					r.endDate !== range.endDate ||
					r.startTime !== range.startTime ||
					r.endTime !== range.endTime
				);
			}

			// Add to beginning and limit to MAX_RECENT_ITEMS
			recentTimeRanges = [newRange, ...recentTimeRanges].slice(0, MAX_RECENT_ITEMS);

			localStorage.setItem(RECENT_TIME_RANGES_KEY, JSON.stringify(recentTimeRanges));
		} catch (e) {
		}
	}

	/** @param {number} index */
	function removeRecentTimeRange(index) {
		recentTimeRanges = recentTimeRanges.filter((_, i) => i !== index);
		try {
			localStorage.setItem(RECENT_TIME_RANGES_KEY, JSON.stringify(recentTimeRanges));
		} catch (e) {
		}
	}

	/** @param {string} optionValue */
	function handleRelativeOptionClick(/** @type {string} */ optionValue) {
		timeRange = optionValue;

		const option = TIME_RANGE_OPTIONS.find(x => x.value === optionValue);

		if (option) {
			saveRecentTimeRange({
				startDate: '',
				endDate: '',
				startTime: '',
				endTime: '',
				timeRange: optionValue,
				label: option.label
			});
		}

		startDate = '';
		endDate = '';
		startTime = '';
		endTime = '';
		showDatePicker = false;
		dispatch('change', { timeRange, startDate, endDate, startTime, endTime });
	}

	function clearSelection() {
		timeRange = '';
		startDate = '';
		endDate = '';
		startTime = '';
		endTime = '';
		showDatePicker = false;
		dispatch('change', { timeRange, startDate, endDate, startTime, endTime });
	}

	/** @param {any} range */
	function handleRecentOptionClick(range) {
		if (range.timeRange) {
			timeRange = range.timeRange;
		} else {
			timeRange = CUSTOM_DATE_RANGE;
			startDate = range.startDate;
			endDate = range.endDate;
			startTime = range.startTime;
			endTime = range.endTime;
		}
		showDatePicker = false;
		dispatch('change', { timeRange, startDate, endDate, startTime, endTime });
	}

	function handleCustomConfirm() {
		if (validationError) {
			return;
		}

		if (startDate) {
			if (!endDate) {
				endDate = startDate;
			}

			if (!startTime) {
				startTime = DEFAULT_START_TIME;
			}
			if (!endTime) {
				endTime = DEFAULT_END_TIME;
			}

			timeRange = CUSTOM_DATE_RANGE;
			saveRecentTimeRange({ startDate, endDate, startTime, endTime });
		}
		showDatePicker = false;
		dispatch('change', { timeRange, startDate, endDate, startTime, endTime });
	}


	function toggleDropdown() {
		showDatePicker = !showDatePicker;
		if (showDatePicker) {
			datePickerTab = timeRange === CUSTOM_DATE_RANGE ? TAB_CUSTOM : TAB_RELATIVE;
		}
	}

	/** @param {any} e */
	function handleClickOutside(e) {
		e.preventDefault();

		const curNode = e.detail.currentNode;
		const targetNode = e.detail.targetNode;

		if (!curNode?.contains(targetNode)) {
			showDatePicker = false;
		}
	}
</script>

<div class="multiselect-container" use:clickoutsideDirective on:clickoutside={handleClickOutside}>
	<!-- svelte-ignore a11y-click-events-have-key-events -->
	<!-- svelte-ignore a11y-no-static-element-interactions -->
	<div class="display-container" on:click={toggleDropdown} id={TOOLTIP_ID}>
		<Input
			type="text"
			class="clickable"
			value={timeRangeDisplayText}
			placeholder="Select time range"
			readonly
		/>
		<div class="display-suffix {showDatePicker ? 'show-list' : ''}">
			<i class="bx bx-chevron-down" />
		</div>
	</div>
	{#if timeRange === CUSTOM_DATE_RANGE && startDate && endDate}
		<BotsharpTooltip
			containerClasses="time-range-tooltip"
			target={TOOLTIP_ID}
			placement="bottom"
		>
			<div>
				<div>{formatDateForDisplay(startDate)} {formatTimeTo12Hour(startTime || DEFAULT_START_TIME)}</div>
				<div>{formatDateForDisplay(endDate)} {formatTimeTo12Hour(endTime || DEFAULT_END_TIME)}</div>
			</div>
		</BotsharpTooltip>
	{/if}
	{#if showDatePicker}
		<ul class="option-list time-range-dropdown" style="min-width: 350px; max-width: 380px;">
			<div class="time-range-content">
			<ul class="nav nav-tabs border-bottom mb-0 px-2 pt-2" role="tablist">
				<li class="nav-item flex-fill" role="presentation">
					<button
						class="nav-link fw-semibold {datePickerTab === TAB_RELATIVE ? 'active text-primary' : 'text-muted'}"
						type="button"
						role="tab"
						style="padding: 0.5rem 0.75rem; {datePickerTab === TAB_RELATIVE ? 'border-bottom: 2px solid var(--bs-primary) !important; margin-bottom: -1px;' : ''}"
						on:click={() => datePickerTab = TAB_RELATIVE}
					>
						Relative
					</button>
				</li>
				<li class="nav-item flex-fill" role="presentation">
					<button
						class="nav-link fw-semibold {datePickerTab === TAB_RECENT ? 'active text-primary' : 'text-muted'}"
						type="button"
						role="tab"
						style="padding: 0.5rem 0.75rem; {datePickerTab === TAB_RECENT ? 'border-bottom: 2px solid var(--bs-primary) !important; margin-bottom: -1px;' : ''}"
						on:click={() => datePickerTab = TAB_RECENT}
					>
						Recent
					</button>
				</li>
				<li class="nav-item flex-fill" role="presentation">
					<button
						class="nav-link fw-semibold {datePickerTab === TAB_CUSTOM ? 'active text-primary' : 'text-muted'}"
						type="button"
						role="tab"
						style="padding: 0.5rem 0.75rem; {datePickerTab === TAB_CUSTOM ? 'border-bottom: 2px solid var(--bs-primary) !important; margin-bottom: -1px;' : ''}"
						on:click={() => {
							datePickerTab = TAB_CUSTOM;
							// Set default dates to yesterday and today if not already set
							if (!startDate && !endDate) {
								startDate = getYesterdayStr();
								endDate = getTodayStr();
							}
						}}
					>
						Custom
					</button>
				</li>
			</ul>

			<div class="p-2">
				{#if datePickerTab === TAB_RELATIVE}
					<!-- svelte-ignore a11y-click-events-have-key-events -->
					<!-- svelte-ignore a11y-no-static-element-interactions -->
					<div
						class="option-item clickable justify-content-center"
						role="button"
						tabindex="0"
						on:click|preventDefault|stopPropagation={() => {
							clearSelection();
						}}
					>
						<div class="line-align-center text-secondary">
							{`Clear selection`}
						</div>
					</div>
					{#each presetTimeRangeOptions as option}
						<!-- svelte-ignore a11y-click-events-have-key-events -->
						<!-- svelte-ignore a11y-no-static-element-interactions -->
						<div
							class="option-item clickable"
							role="button"
							tabindex="0"
							on:click|preventDefault|stopPropagation={() => {
								handleRelativeOptionClick(option.value);
							}}
						>
							<div class="line-align-center select-box">
								{#if timeRange === option.value}
									<i class="bx bx-check text-primary" />
								{:else}
									{' '}
								{/if}
							</div>
							<div class="line-align-center select-name">
								{option.label}
							</div>
						</div>
					{/each}
				{:else if datePickerTab === TAB_RECENT}
					{#if recentTimeRanges.length === 0}
						<div class="option-item">
							<div class="text-muted text-center py-3 w-100">
								<i class="bx bx-time-five mb-2" style="font-size: 1.5rem;"></i>
								<p class="mb-0 small">{'No recent time ranges'}</p>
								<!-- <p class="mb-0 small text-muted">{EMPTY_RECENT_HINT}</p> -->
							</div>
						</div>
					{:else}
						{#each recentTimeRanges as range, index}
							<!-- svelte-ignore a11y-click-events-have-key-events -->
							<!-- svelte-ignore a11y-no-static-element-interactions -->
							<div
								class="option-item clickable d-flex justify-content-between"
								role="button"
								tabindex="0"
								on:click|preventDefault|stopPropagation={() => {
									handleRecentOptionClick(range);
								}}
							>
								<div class="line-align-center select-name" style="font-size: 12px;">
									{range.label}
								</div>
								<button
									type="button"
									class="btn btn-sm btn-link text-muted p-0"
									title="Remove from recent"
									on:click|preventDefault|stopPropagation={() => {
										removeRecentTimeRange(index);
									}}
								>
									<i class="bx bx-x"></i>
								</button>
							</div>
						{/each}
					{/if}
				{:else if datePickerTab === TAB_CUSTOM}
					<div class="row mb-2">
						<div class="col-7">
							<label for="start-date-picker" class="form-label mb-1">From Date:</label>
							<Input
								id="start-date-picker"
								type="date"
								style="font-size: 14px;"
								bind:value={startDate}
								class="form-control"
							/>
						</div>
						<div class="col-5">
							<label for="start-time-picker" class="form-label mb-1">Time:</label>
							<Input
								id="start-time-picker"
								type="time"
								bind:value={startTime}
								class="form-control"
							/>
						</div>
					</div>
					<div class="row mb-2c">
						<div class="col-7">
							<label for="end-date-picker" class="form-label mb-1">To Date:</label>
							<Input
								id="end-date-picker"
								type="date"
								bind:value={endDate}
								class="form-control"
							/>
						</div>
						<div class="col-5">
							<label for="end-time-picker" class="form-label mb-1">Time:</label>
							<Input
								id="end-time-picker"
								type="time"
								bind:value={endTime}
								class="form-control"
							/>
						</div>
					</div>
					{#if validationError}
						<div class="alert alert-danger py-2 px-3 small mb-3">
							{validationError}
						</div>
					{/if}
					<div class="d-flex justify-content-end gap-2 mt-3"
						 style="margin-right: 12px;">
						<Button
							color="primary"
							type="button"
							disabled={!!validationError}
							on:click={(e) => {
								e.preventDefault();
								e.stopPropagation();
								handleCustomConfirm();
							}}
						>
							Apply
						</Button>
					</div>
				{/if}
			</div>
		</ul>
	{/if}
</div>
