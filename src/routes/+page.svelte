<script lang="ts">
	import { PUBLIC_BACK_URL } from '$env/static/public';
	import {
		createCalendar,
		createViewDay,
		createViewMonthAgenda,
		createViewMonthGrid,
		createViewWeek
	} from '@schedule-x/calendar';
	import { createEventsServicePlugin } from '@schedule-x/events-service';
	import { ScheduleXCalendar } from '@schedule-x/svelte';
	import '@schedule-x/theme-shadcn/dist/index.css';
	import { onMount } from 'svelte';
	import { Toaster, toast } from 'svelte-sonner';

	let calendarApp = $state(null);
	let eventsService = $state(null);
	let events = $state([]);
	let isLoading = $state(false);
	let userData = $state('');
	let isAccordionOpen = $state(false);
	let help = `{
  "courses": ["Course1", "Course2", ...],       // Array of course names
  "examDates": {                                // Object mapping courses to exam dates
    "Course1": "YYYY-MM-DD",
    "Course2": "YYYY-MM-DD"
  },
  "projects": {                                 // Optional - Project deadlines
    "ProjectName": "YYYY-MM-DD"                 // Simple format
    // OR with intensity
    "ProjectName": {
      "deadline": "YYYY-MM-DD",
      "intensity": 1-5                          // 1=low, 5=high intensity
    }
  },
  "currentEvents": [                            // Academic events
    {
      "title": "Event Name",
      "date": "YYYY-MM-DD",
      "start": "HH:MM",
      "end": "HH:MM"
    }
  ],
  "commitments": [                              // Personal commitments
    {
      "title": "Commitment Name",
      "date": "YYYY-MM-DD",
      "start": "HH:MM",
      "end": "HH:MM"
    }
  ],
  "persona": {                                  // Optional - Student personality/preferences
    "type": "party-goer",                       // Options: "party-goer", "early-bird", "night-owl", etc.
    "preferredStudyHours": {
      "start": "HH:MM",
      "end": "HH:MM"
    },
    "preferredFreeDays": ["Friday", "Saturday"],
    "studyPreferences": {
      "preferredSessionLength": "medium",       // "short", "medium", "long"
      "maximumSessionsPerDay": 3
    },
    "customActivities": [                       // Regular weekly activities
      {
        "title": "Activity Name",
        "dayOfWeek": "Monday",                  // Day of week this occurs
        "start": "HH:MM",
        "end": "HH:MM"
      }
    ]
  }
}`;

	function formatDuration(durationString) {
		if (durationString && typeof durationString === 'string') {
			if (durationString.includes('hours') || durationString.includes('hour')) {
				const hours = parseFloat(durationString.split(' ')[0]);
				return hours * 60;
			} else if (durationString.includes('minutes') || durationString.includes('minute')) {
				return parseInt(durationString.split(' ')[0]);
			}
			return parseInt(durationString);
		}
		return 60;
	}

	function calculateEndTime(date, startTime, duration) {
		const startDate = new Date(`${date}T${startTime}:00`);
		const durationMinutes = formatDuration(duration);
		const endDate = new Date(startDate.getTime() + durationMinutes * 60 * 1000);
		const endTimeString = `${endDate.getHours().toString().padStart(2, '0')}:${endDate.getMinutes().toString().padStart(2, '0')}`;
		return `${date} ${endTimeString}`;
	}

	function convertToScheduleXEvents(studyEvents) {
		if (!studyEvents || !Array.isArray(studyEvents)) return [];

		return studyEvents.map((event, index) => {
			try {
				const isFullDayEvent =
					event.type === 'exam' ||
					event.type === 'project' ||
					(event.start === event.date && event.end === event.date);

				let startDateTime, endDateTime;

				if (isFullDayEvent) {
					startDateTime = event.date;
					endDateTime = event.date;
				} else {
					const startTime = event.startTime || event.start;

					if (!startTime || typeof startTime !== 'string') {
						console.error('Invalid start time for event:', event);
						startDateTime = `${event.date} 09:00`;
					} else {
						startDateTime = `${event.date} ${startTime}`;
					}

					if (event.end) {
						endDateTime = `${event.date} ${event.end}`;
					} else if (event.duration) {
						endDateTime = calculateEndTime(
							event.date,
							event.startTime || event.start,
							event.duration
						);
					} else {
						const startParts = (event.startTime || event.start).split(':');
						const endHour = parseInt(startParts[0]) + 1;
						endDateTime = `${event.date} ${endHour.toString().padStart(2, '0')}:${startParts[1]}`;
					}
				}

				let additionalClasses = ['study-event'];
				if (event.type === 'current-event') {
					additionalClasses = ['current-event'];
				} else if (event.type === 'commitment') {
					additionalClasses = ['commitment-event'];
				} else if (event.type === 'exam') {
					additionalClasses = ['exam-event'];
				} else if (event.type === 'project') {
					additionalClasses = ['project-event'];
				}

				return {
					id: `event-${index + 1}`,
					title: event.course || event.title,
					start: startDateTime,
					end: endDateTime,
					description: event.goals ? event.goals.join('\n') : '',
					_options: {
						additionalClasses: additionalClasses
					}
				};
			} catch (error) {
				console.error('Error converting event:', event, error);
				return {
					id: `event-${index + 1}`,
					title: event.course || event.title || 'Untitled Event',
					start: event.date ? `${event.date} 09:00` : '2025-03-13 09:00',
					end: event.date ? `${event.date} 10:00` : '2025-03-13 10:00',
					_options: {
						additionalClasses: ['error-event']
					}
				};
			}
		});
	}

	function toggleAccordion() {
		isAccordionOpen = !isAccordionOpen;
	}

	function validateUserData() {
		try {
			JSON.parse(userData);
			return true;
		} catch (error) {
			toast.error('Invalid JSON format: ' + error.message);
			return false;
		}
	}

	async function loadPlanning() {
		if (!validateUserData()) return;

		isLoading = true;

		const planningPromise = new Promise(async (resolve, reject) => {
			try {
				const userDataObj = JSON.parse(userData);

				const response = await fetch(`${PUBLIC_BACK_URL}/api/planning`, {
					method: 'POST',
					headers: {
						'Content-Type': 'application/json'
					},
					body: JSON.stringify(userDataObj)
				});

				if (!response.ok) {
					throw new Error(`API responded with status: ${response.status}`);
				}

				const data = await response.json();
				resolve(data);
			} catch (err) {
				console.error('Error loading planning:', err);
				reject(err);
			}
		});

		toast.promise(planningPromise, {
			loading: 'Generating study planning...',
			success: (data) => {
				console.log('API Response:', data);
				events = data.events || [];
				updateCalendar();

				const studySessions = data.validPlanning?.length || 0;
				const examCount = events.filter((e) => e.type === 'exam').length;
				const projectCount = events.filter((e) => e.type === 'project').length;
				const otherEvents = events.length - studySessions - examCount - projectCount;

				return `Loaded ${studySessions} study sessions, ${examCount} exams, ${projectCount} projects, and ${otherEvents} other events`;
			},
			error: (err) => {
				events = [];
				return `Error: ${err.message}`;
			},
			finally: () => {
				isLoading = false;
			}
		});
	}

	function updateCalendar() {
		try {
			const calendarEvents = convertToScheduleXEvents(events);
			console.log('Calendar events:', calendarEvents);

			if (calendarApp && calendarApp.eventsService) {
				calendarApp.eventsService.set(calendarEvents);
			} else {
				console.warn('Calendar or events service not initialized');
			}
		} catch (error) {
			console.error('Error updating calendar:', error);
			toast.error('Failed to update calendar: ' + error.message);
		}
	}

	onMount(async () => {
		// Initialize calendar
		const eventsServicePlugin = createEventsServicePlugin();
		calendarApp = createCalendar({
			views: [createViewDay(), createViewWeek(), createViewMonthGrid(), createViewMonthAgenda()],
			locale: 'fr-FR',
			theme: 'shadcn',
			events: [],
			defaultView: 'week',
			plugins: [eventsServicePlugin]
		});

		eventsService = calendarApp.eventsService;

		try {
			const response = await fetch('/data.json');
			if (response.ok) {
				const data = await response.json();
				userData = JSON.stringify(data, null, 2);
			} else {
				console.error('Failed to load initial data');
				userData = JSON.stringify(
					{
						courses: ['Mathematics', 'History', 'Physics'],
						examDates: {
							Mathematics: '2025-03-17',
							History: '2025-03-20',
							Physics: '2025-03-24'
						},
						currentEvents: [],
						commitments: []
					},
					null,
					2
				);
			}
		} catch (error) {
			console.error('Error loading initial data:', error);
		}
	});
</script>

<Toaster position="bottom-right" />

<div class="mx-auto flex flex-col gap-5 p-8">
	<h1 class="mb-2 text-2xl font-bold">Study Planning Generator</h1>

	<div class="mb-2">
		<button
			onclick={toggleAccordion}
			class="flex w-full items-center justify-between rounded-md bg-gray-100 px-4 py-2 transition-colors hover:bg-gray-200"
		>
			<span class="font-medium">Input Format Documentation (expand)</span>
			<span>{isAccordionOpen ? '−' : '+'}</span>
		</button>

		{#if isAccordionOpen}
			<div class="mt-2 rounded-md border border-gray-200 bg-gray-50 p-4">
				<h3 class="mb-2 font-semibold">Required Input Format:</h3>
				<p class="mb-2">The input must be a valid JSON object with the following structure:</p>

				<div class="mb-2">
					<textarea
						id="userData"
						bind:value={help}
						disabled
						rows="15"
						class="w-full rounded-md border border-gray-300 p-2 font-mono text-sm"
						spellcheck="false"
					></textarea>
				</div>

				<p class="text-sm text-gray-600">
					Note: All dates should be in YYYY-MM-DD format and times in 24-hour HH:MM format.
				</p>
			</div>
		{/if}
	</div>

	<div class="mb-2">
		<label for="userData" class="mb-1 block text-sm font-medium">User Data (JSON):</label>
		<textarea
			id="userData"
			bind:value={userData}
			rows="15"
			class="w-full rounded-md border border-gray-300 p-2 font-mono text-sm"
			spellcheck="false"
		></textarea>
	</div>

	<div class="mt-8 flex flex-col gap-3">
		<button
			onclick={loadPlanning}
			disabled={isLoading}
			class="rounded bg-red-200 px-6 py-3 font-bold transition-colors hover:bg-red-300 disabled:bg-gray-200"
		>
			{isLoading ? 'Generating, please wait...' : 'Generate Study Planning'}
		</button>

		<p class="text-sm text-red-300">
			⚠️ Generation may take up to a minute. Please be patient after clicking the button.
		</p>

		<span class="-mt-2 text-sm text-gray-400">
			(Please check your persona and prefered free daays if you see any gaps)
		</span>
	</div>

	<div class="mt-4 w-full">
		{#if calendarApp}
			<ScheduleXCalendar {calendarApp} />
		{:else}
			<p class="py-10 text-center text-gray-500">Loading calendar...</p>
		{/if}
	</div>
</div>

<style>
	:global(.study-event) {
		background: rgba(203, 241, 170, 0.9) !important; /* Pastel green */
		color: black !important;
		border: none !important;
	}
	:global(.current-event) {
		background: rgba(173, 216, 230, 0.9) !important; /* Pastel blue */
		color: black !important;
		border: none !important;
	}
	:global(.commitment-event) {
		background: rgba(255, 182, 193, 0.9) !important; /* Pastel pink */
		color: black !important;
		border: none !important;
	}
	:global(.exam-event) {
		background: rgba(255, 218, 185, 0.9) !important; /* Pastel peach */
		color: black !important;
		border: none !important;
	}
	:global(.project-event) {
		background: rgba(216, 191, 216, 0.9) !important; /* Pastel purple */
		color: black !important;
		border: none !important;
	}
	:global(.error-event) {
		background: rgba(211, 211, 211, 0.5) !important; /* Light gray */
		border: 2px dashed #ffb6c1 !important; /* Light pink border */
	}
</style>
