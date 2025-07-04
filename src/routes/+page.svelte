<script lang="ts">
	import {
		extendByRecurrenceRule,
		type IcsEvent
	} from 'ts-ics';
	import type { PageData } from './$types';

	interface CalendarData {
		name: string;
		events: IcsEvent[];
	}

	let { data }: { data: PageData } = $props();
	
	console.log('Page data:', data);
	let calendarData = $derived(data?.calendarData || []);

	// Generate 7-day rolling window with today as second column
	function getDateRange(): Date[] {
		const dates: Date[] = [];
		const today = new Date();

		for (let i = -1; i <= 5; i++) {
			// Create date at local midnight to avoid timezone issues
			const targetDate = new Date(today);
			targetDate.setDate(today.getDate() + i);
			const date = new Date(targetDate.getFullYear(), targetDate.getMonth(), targetDate.getDate());
			dates.push(date);
		}

		return dates;
	}

	const dateRange = getDateRange();

	function formatDate(date: Date): string {
		return date.toLocaleDateString('en-US', {
			weekday: 'short',
			month: 'short',
			day: 'numeric'
		});
	}

	interface EventRectangle {
		event: IcsEvent;
		x: number; // starting column (0-6)
		y: number; // top position in pixels
		width: number; // number of columns spanned
		height: number; // height in pixels
		isPartial: boolean;
		partialType?: 'start' | 'end' | 'both';
	}

	interface ColumnSpace {
		column: number;
		occupiedRanges: Array<{ start: number; end: number }>;
	}

	function expandRecurringEvents(
		events: IcsEvent[],
		windowStart: Date,
		windowEnd: Date
	): IcsEvent[] {
		const expandedEvents: IcsEvent[] = [];

		events.forEach((event) => {
			if (event.recurrenceRule) {
				try {
					console.log('Processing recurring event:', event.summary, 'Rule:', event.recurrenceRule);

					// Use the original event's start date as the recurrence base
					const originalEventStart = new Date(event.start.date);

					// Extend our search range significantly to ensure we catch all instances
					const expandedWindowStart = new Date(windowStart);
					expandedWindowStart.setDate(expandedWindowStart.getDate() - 7); // 1 week before
					const expandedWindowEnd = new Date(windowEnd);
					expandedWindowEnd.setDate(expandedWindowEnd.getDate() + 7); // 1 week after

					const recurringDates = extendByRecurrenceRule(event.recurrenceRule, {
						start: expandedWindowStart,
						end: expandedWindowEnd
					});

					console.log(`Found ${recurringDates.length} recurring instances for ${event.summary}`);

					// Create individual event instances for each occurrence
					recurringDates.forEach((occurrenceDate, index) => {
						// Preserve the original time of day from the base event
						const baseEventTime = new Date(event.start.date);

						// Create new event start using the occurrence date but preserving original time
						// Use local timezone construction instead of copying the Date object
						const newEventStart = new Date(
							occurrenceDate.getFullYear(),
							occurrenceDate.getMonth(),
							occurrenceDate.getDate(),
							baseEventTime.getHours(),
							baseEventTime.getMinutes(),
							baseEventTime.getSeconds(),
							baseEventTime.getMilliseconds()
						);

						// Calculate end time based on original duration
						let newEventEnd: Date | undefined;
						if (event.end) {
							const originalDuration =
								new Date(event.end.date).getTime() - new Date(event.start.date).getTime();
							newEventEnd = new Date(newEventStart.getTime() + originalDuration);
						}

						// Create a new event instance for this occurrence
						const recurringEventInstance: IcsEvent = {
							...event,
							start: {
								...event.start,
								date: newEventStart
							},
							end: newEventEnd
								? {
										...event.end!,
										date: newEventEnd
									}
								: event.end,
							// Add a unique identifier for this instance
							uid: `${event.uid}_${index}`,
							// Remove recurrence rule from instances to prevent re-expansion
							recurrenceRule: undefined
						};

						expandedEvents.push(recurringEventInstance);
					});
				} catch (error) {
					console.warn('Failed to expand recurring event:', event.summary, error);
					// Fall back to original event if recurrence expansion fails
					expandedEvents.push(event);
				}
			} else {
				// Non-recurring event - include as is
				expandedEvents.push(event);
			}
		});

		console.log(`Expanded ${events.length} events to ${expandedEvents.length} total events`);
		return expandedEvents;
	}

	function getEventsInWindow(events: IcsEvent[], windowStart: Date, windowEnd: Date): IcsEvent[] {
		// First expand all recurring events
		const expandedEvents = expandRecurringEvents(events, windowStart, windowEnd);

		// Filter to events within reasonable time range AND our 7-day window
		// Since we're in 2025, be more restrictive about old events
		const filterStart = new Date(windowStart);
		filterStart.setMonth(filterStart.getMonth() - 2); // 2 months before window
		const filterEnd = new Date(windowStart);
		filterEnd.setMonth(filterEnd.getMonth() + 3); // 3 months after window start

		const filteredEvents = expandedEvents.filter((event) => {
			const eventStart = new Date(event.start.date);
			const eventEnd = event.end ? new Date(event.end.date) : eventStart;

			// Skip very old or very future events
			if (eventEnd < filterStart || eventStart > filterEnd) {
				return false;
			}

			// Check if event intersects with our exact 7-day window
			// Normalize dates to midnight using LOCAL timezone, not UTC
			const eventDateOnly = new Date(
				eventStart.getFullYear(),
				eventStart.getMonth(),
				eventStart.getDate()
			);

			const windowStartOnly = new Date(
				windowStart.getFullYear(),
				windowStart.getMonth(),
				windowStart.getDate()
			);

			const windowEndOnly = new Date(
				windowEnd.getFullYear(),
				windowEnd.getMonth(),
				windowEnd.getDate()
			);

			if (isAllDayEvent(event)) {
				// For all-day events, end date is exclusive
				const eventEndOnly = new Date(
					eventEnd.getFullYear(),
					eventEnd.getMonth(),
					eventEnd.getDate()
				);
				return eventDateOnly < windowEndOnly && eventEndOnly > windowStartOnly;
			} else {
				// For timed events, check if the event date falls within our window
				return eventDateOnly >= windowStartOnly && eventDateOnly < windowEndOnly;
			}
		});

		// Debug: Log events by day to see distribution
		const eventsByDay: { [key: string]: number } = {};
		const windowDates: string[] = [];
		for (let i = 0; i < 7; i++) {
			const date = new Date(windowStart);
			date.setDate(date.getDate() + i);
			windowDates.push(date.toDateString());
		}

		console.log('Expected window dates:', windowDates);

		filteredEvents.forEach((event) => {
			const eventDate = new Date(event.start.date);
			const dayKey = eventDate.toDateString();
			eventsByDay[dayKey] = (eventsByDay[dayKey] || 0) + 1;
		});
		console.log('Events by day (after filtering):', eventsByDay);
		console.log(`Filtered to ${filteredEvents.length} events in the 7-day window`);

		return filteredEvents;
	}

	function getEventHeight(
		event: IcsEvent,
		isPartial: boolean,
		partialType?: string,
		width: number = 1
	): number {
		// Height levels for event sizing
		const HEIGHT_COMPACT = 32; // Very compact for multi-day events
		const HEIGHT_SMALL = 45; // Standard small
		const HEIGHT_MEDIUM = 62; // Medium for longer content

		// Calculate text content length
		let text = event.summary || '';
		if (!isAllDayEvent(event) && !isPartial) {
			text = `${formatEventTime(event)} ${text}`;
		}
		if (isPartial) {
			text = `← ${text} →`; // Add arrows
		}

		// Different sizing strategy based on event type and width
		if (width > 1) {
			// Multi-day events: prioritize compactness
			if (text.length <= 30) {
				return HEIGHT_COMPACT; // Very compact for short multi-day
			} else if (text.length <= 60) {
				return HEIGHT_SMALL; // Still compact for medium multi-day
			} else {
				return HEIGHT_MEDIUM; // Only go to medium for very long multi-day
			}
		} else {
			// Single-day events: optimize for single-line content
			if (isAllDayEvent(event)) {
				// All-day single events - most can be compact
				if (text.length <= 20) {
					return HEIGHT_COMPACT;
				} else if (text.length <= 40) {
					return HEIGHT_SMALL;
				} else {
					return HEIGHT_MEDIUM;
				}
			} else {
				// Timed events (time + text on same line now)
				if (text.length <= 25) {
					return HEIGHT_COMPACT; // Compact for short timed events
				} else if (text.length <= 45) {
					return HEIGHT_SMALL; // Small for medium timed events
				} else {
					return HEIGHT_MEDIUM; // Medium for longer timed events
				}
			}
		}
	}

	function getEventPriority(event: IcsEvent, width: number): number {
		let priority = 0;

		// Multi-day events get highest priority
		if (width > 1) priority += 1000;

		// All-day events get higher priority than timed events
		if (isAllDayEvent(event)) priority += 100;

		// Earlier events get slight preference (chronological order)
		const eventTime = new Date(event.start.date).getTime();
		const dayStart = new Date(event.start.date);
		dayStart.setHours(0, 0, 0, 0);
		const timeOfDay = eventTime - dayStart.getTime();
		priority += Math.max(0, 24 - timeOfDay / (1000 * 60 * 60)); // Earlier = higher priority

		return priority;
	}

	function calculateColumnLoad(columns: ColumnSpace[]): number[] {
		return columns.map((col) => {
			return col.occupiedRanges.reduce((total, range) => {
				return total + (range.end - range.start);
			}, 0);
		});
	}

	function findOptimalPosition(
		columns: ColumnSpace[],
		x: number,
		width: number,
		height: number,
		columnLoads: number[]
	): { x: number; y: number } {
		let bestX = x;
		let bestY = 8; // Start with padding from top
		const gap = 4; // Gap between events

		// Events must stay in their correct day column, but optimize vertical position
		bestY = findLowestPositionInColumn(columns, x, width, height);

		return { x: bestX, y: bestY };
	}

	function findLowestPositionInColumn(
		columns: ColumnSpace[],
		col: number,
		width: number,
		height: number
	): number {
		let y = 4; // Start with padding from top
		const gap = 2; // Gap between events
		let attempts = 0; // Safety counter
		const maxAttempts = 100; // Prevent infinite loops

		// Use the original proven algorithm with safety check
		while (attempts < maxAttempts) {
			let hasConflict = false;

			for (let c = col; c < col + width && c < columns.length; c++) {
				const column = columns[c];
				for (const range of column.occupiedRanges) {
					if (!(y + height <= range.start || y >= range.end)) {
						hasConflict = true;
						y = Math.max(y, range.end + gap);
						break;
					}
				}
				if (hasConflict) break;
			}

			if (!hasConflict) {
				return y;
			}

			attempts++;
		}

		// Fallback: if we hit max attempts, return a safe position
		console.warn('Hit max attempts in collision detection, using fallback position');
		return y + attempts * 60; // Space events out vertically as fallback
	}

	function occupySpace(
		columns: ColumnSpace[],
		x: number,
		width: number,
		y: number,
		height: number
	) {
		for (let col = x; col < x + width; col++) {
			if (col >= columns.length) continue;

			columns[col].occupiedRanges.push({
				start: y,
				end: y + height
			});

			// Sort ranges to help with future collision detection
			columns[col].occupiedRanges.sort((a, b) => a.start - b.start);
		}
	}

	function layoutCalendarEvents(events: IcsEvent[], windowStart: Date): EventRectangle[] {
		const windowEnd = new Date(windowStart);
		windowEnd.setDate(windowEnd.getDate() + 7);

		const relevantEvents = getEventsInWindow(events, windowStart, windowEnd);
		const rectangles: EventRectangle[] = [];

		// Initialize column space tracking (7 columns for 7 days)
		const columns: ColumnSpace[] = Array.from({ length: 7 }, (_, i) => ({
			column: i,
			occupiedRanges: []
		}));

		// Prepare events with metadata for sorting and processing
		const eventData = relevantEvents.map((event) => {
			const eventStart = new Date(event.start.date);
			let eventEnd = event.end ? new Date(event.end.date) : new Date(eventStart);

			// For single-day events without end time, make them end on the same day
			if (!event.end) {
				eventEnd = new Date(eventStart);
				if (isAllDayEvent(event)) {
					eventEnd.setDate(eventEnd.getDate() + 1);
				}
			}

			// Calculate position within window using date comparison, not time difference
			// This avoids timezone issues entirely by comparing year/month/date values directly
			const windowStartDate = windowStart.getDate();
			const windowStartMonth = windowStart.getMonth();
			const windowStartYear = windowStart.getFullYear();

			const eventStartDate = eventStart.getDate();
			const eventStartMonth = eventStart.getMonth();
			const eventStartYear = eventStart.getFullYear();

			const eventEndDate = eventEnd.getDate();
			const eventEndMonth = eventEnd.getMonth();
			const eventEndYear = eventEnd.getFullYear();

			// Calculate day difference by creating comparable date values
			const windowStartComparable =
				windowStartYear * 10000 + windowStartMonth * 100 + windowStartDate;
			const eventStartComparable = eventStartYear * 10000 + eventStartMonth * 100 + eventStartDate;
			const eventEndComparable = eventEndYear * 10000 + eventEndMonth * 100 + eventEndDate;

			// Convert to simple day difference (assuming events are within a reasonable time range)
			let startColumn = 0;
			let endColumn = 0;

			// Calculate days from window start to event start
			const windowDate = new Date(windowStartYear, windowStartMonth, windowStartDate);
			const eventDate = new Date(eventStartYear, eventStartMonth, eventStartDate);
			const eventEndDateObj = new Date(eventEndYear, eventEndMonth, eventEndDate);

			startColumn = Math.round(
				(eventDate.getTime() - windowDate.getTime()) / (1000 * 60 * 60 * 24)
			);
			endColumn = Math.round(
				(eventEndDateObj.getTime() - windowDate.getTime()) / (1000 * 60 * 60 * 24)
			);

			// Minimal debug for key events
			if (event.summary.toLowerCase().includes('koffee')) {
				console.log(`${event.summary}: ${eventStart.toDateString()} → column ${startColumn}`);
				console.log(`Window start: ${windowStart.toDateString()}`);
				console.log(`Expected: column 1 for Monday (today), got column ${startColumn}`);
			}

			// Handle all-day events and column boundaries
			if (isAllDayEvent(event)) {
				endColumn = endColumn - 1;
			}

			if (!isAllDayEvent(event) && eventDate.getTime() === eventEndDateObj.getTime()) {
				endColumn = startColumn;
			}

			// Check partial events
			let isPartial = false;
			let partialType: 'start' | 'end' | 'both' | undefined;

			if (startColumn < 0) {
				isPartial = true;
				partialType = 'start';
				startColumn = 0;
			}

			if (endColumn > 6) {
				isPartial = true;
				partialType = partialType === 'start' ? 'both' : 'end';
				endColumn = 6;
			}

			const width = Math.max(1, endColumn - startColumn + 1);
			const height = getEventHeight(event, isPartial, partialType, width);
			const priority = getEventPriority(event, width);

			return {
				event,
				startColumn,
				width,
				height,
				isPartial,
				partialType,
				priority
			};
		});

		// Sort by priority (multi-day first, then by other factors)
		eventData.sort((a, b) => b.priority - a.priority);

		// TWO-PASS ALGORITHM

		// PASS 1: Place multi-day events first (they have more constraints)
		const multiDayEvents = eventData.filter((ed) => ed.width > 1);
		const singleDayEvents = eventData.filter((ed) => ed.width === 1);

		for (const eventData of multiDayEvents) {
			const columnLoads = calculateColumnLoad(columns);
			const position = findOptimalPosition(
				columns,
				eventData.startColumn,
				eventData.width,
				eventData.height,
				columnLoads
			);

			const rectangle: EventRectangle = {
				event: eventData.event,
				x: position.x,
				y: position.y,
				width: eventData.width,
				height: eventData.height,
				isPartial: eventData.isPartial,
				partialType: eventData.partialType
			};

			occupySpace(columns, position.x, eventData.width, position.y, eventData.height);
			rectangles.push(rectangle);
		}

		// PASS 2: Place single-day events in remaining space
		for (const eventData of singleDayEvents) {
			const columnLoads = calculateColumnLoad(columns);
			const position = findOptimalPosition(
				columns,
				eventData.startColumn,
				eventData.width,
				eventData.height,
				columnLoads
			);

			const rectangle: EventRectangle = {
				event: eventData.event,
				x: position.x,
				y: position.y,
				width: eventData.width,
				height: eventData.height,
				isPartial: eventData.isPartial,
				partialType: eventData.partialType
			};

			occupySpace(columns, position.x, eventData.width, position.y, eventData.height);
			rectangles.push(rectangle);
		}

		return rectangles;
	}

	function isAllDayEvent(event: IcsEvent): boolean {
		return event.start.type === 'DATE';
	}

	function formatEventTime(event: IcsEvent): string {
		if (isAllDayEvent(event)) return '';
		const time = new Date(event.start.date);
		return time.toLocaleTimeString('en-US', {
			hour: '2-digit',
			minute: '2-digit',
			hour12: false
		});
	}

</script>

<div class="min-h-screen bg-gray-50 p-4">
		<div class="max-w-full overflow-x-auto">
			<!-- Header with dates -->
			<div class="mb-4 flex">
				<div class="w-48 flex-shrink-0"></div>
				{#each dateRange as date}
					<div class="min-w-32 flex-1 px-2 text-center">
						<div class="font-semibold text-gray-800">{formatDate(date)}</div>
						<div class="text-sm text-gray-500">
							{date.getDate() === new Date().getDate() ? 'Today' : ''}
						</div>
					</div>
				{/each}
			</div>

			<!-- Calendar rows -->
			{#each calendarData as calendar}
				{@const eventRectangles = layoutCalendarEvents(calendar.events, dateRange[0])}
				{@const maxHeight = Math.max(
					120,
					Math.max(...eventRectangles.map((r) => r.y + r.height)) + 12
				)}

				<div class="mb-3 rounded-lg border bg-white shadow-sm">
					<div class="flex">
						<!-- Calendar name -->
						<div class="w-48 flex-shrink-0 rounded-l-lg border-r bg-gray-100 p-4">
							<h3 class="text-lg font-bold text-gray-800">{calendar.name}</h3>
						</div>

						<!-- Days container with masonry layout -->
						<div class="relative flex flex-1" style="min-height: {maxHeight}px;">
							{#each dateRange as date, dayIndex}
								<div class="relative flex-1 border-r border-gray-100 last:border-r-0">
									<!-- Column background for visual reference -->
								</div>
							{/each}

							<!-- Events positioned as rectangles -->
							{#each eventRectangles as rect}
								{@const event = rect.event}
								{@const columnWidth = 100 / 7}
								<!-- Percentage width per column -->

								<div
									class="absolute text-sm break-words"
									style="
										left: {rect.isPartial && rect.partialType === 'start'
										? '0'
										: `calc(${rect.x * columnWidth}% + 0.5rem)`};
										top: {rect.y}px;
										width: {rect.isPartial && rect.partialType === 'end'
										? `calc(${rect.width * columnWidth}% + 0.5rem)`
										: rect.isPartial && rect.partialType === 'start'
											? `calc(${rect.width * columnWidth}% - 0.5rem)`
											: rect.isPartial && rect.partialType === 'both'
												? '100%'
												: `calc(${rect.width * columnWidth}% - 1rem)`};
										height: {rect.height}px;
										z-index: 10;
									"
								>
									{#if isAllDayEvent(event)}
										<div
											class="flex h-full flex-col justify-center bg-blue-100 px-2 py-1 text-blue-800 shadow-sm
											{rect.isPartial && rect.partialType === 'start'
												? 'rounded-r border-t border-r border-b'
												: rect.isPartial && rect.partialType === 'end'
													? 'rounded-l border-t border-b border-l'
													: rect.isPartial && rect.partialType === 'both'
														? 'border-t border-b'
														: 'rounded border'}"
										>
											{event.summary}
										</div>
									{:else}
										<div
											class="flex h-full flex-col justify-center bg-orange-100 px-2 py-1 text-orange-800 shadow-sm
											{rect.isPartial && rect.partialType === 'start'
												? 'rounded-r border-t border-r border-b border-l-4 border-l-orange-400'
												: rect.isPartial && rect.partialType === 'end'
													? 'rounded-l border-t border-b border-l border-l-4 border-l-orange-400'
													: rect.isPartial && rect.partialType === 'both'
														? 'border-t border-b border-l-4 border-l-orange-400'
														: 'rounded border-l-4 border-l-orange-400'}"
										>
											<div class="text-xs">
												<span class="font-semibold">{formatEventTime(event)}</span>
												{event.summary}
											</div>
										</div>
									{/if}
								</div>
							{/each}
						</div>
					</div>
				</div>
			{/each}
		</div>
</div>
