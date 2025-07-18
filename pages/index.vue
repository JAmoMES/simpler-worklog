<template>
  <div>
    <div class="mb-4 flex justify-end gap-2">
      <ClientOnly>
        <!-- Fill from Calendar group -->
        <div class="flex items-center gap-1">
          <button
            class="btn btn-primary"
            :disabled="autoFilling || !hasCreds || calendarLoading"
            title="Upload .ics file and configure event mappings with team presets"
            @click="confirmAutoFillFromCalendar"
          >
            <Icon name="lucide:calendar-plus" class="w-4 h-4" />
            Fill from Calendar (.ics)
          </button>
        </div>
        <button
          class="btn btn-primary mr-8"
          title="How to use Fill from Calendar"
          @click="showCalInfo = true"
        >
          <Icon name="lucide:help-circle" class="w-4 h-4" />
        </button>
        <button
          class="btn btn-secondary"
          :disabled="autoFilling || !hasCreds || calendarLoading"
          @click="confirmAutoFill"
        >
          <Icon name="lucide:zap" class="w-4 h-4" />
          Auto-Fill Ceremonies (Sally)
        </button>
        <input
          ref="fileInput"
          type="file"
          accept=".ics"
          style="display: none"
          @change="handleFileSelect"
        />
      </ClientOnly>
    </div>
    <CalendarMonth ref="calRef" @day-selected="onDaySelected" />
    <WorklogModal
      :visible="showWorklog"
      :date="selectedDate"
      @close="showWorklog = false"
    />

    <CeremonyConfigModal
      :visible="showConfigModal"
      @close="showConfigModal = false"
      @saved="onConfigSaved"
    />

    <dialog v-if="autoFilling" class="modal modal-open">
      <div class="modal-box flex flex-col items-center gap-4">
        <progress class="progress progress-primary w-full" />
        <p class="font-semibold">Auto-filling ceremonies…</p>
      </div>
    </dialog>

    <!-- Calendar Info Modal -->
    <dialog v-if="showCalInfo" class="modal modal-open">
      <div class="modal-box max-w-2xl">
        <h3 class="font-bold text-lg mb-2">How to use "Fill from Calendar"</h3>
        <ol class="list-decimal list-inside space-y-3 text-sm">
          <li>
            <strong>Export an .ics file from Google Calendar</strong><br />
            In Google Calendar, hover your calendar →
            <em>Settings &amp; sharing</em> → "Integrate calendar" →
            <em>Export</em>. Un-zip the download to get the *.ics* file.
          </li>
          <li>
            <strong>Click "Fill from Calendar" and select your file</strong
            ><br />
            The app will automatically parse your calendar events, group them by
            title, and open the configuration modal with unique event titles
            pre-filled.
          </li>
          <li>
            <strong>Auto-matching from saved configurations</strong><br />
            If you've used this feature before, the app will automatically fill
            in JIRA issue keys for event titles that match your previously saved
            mappings.
          </li>
          <li>
            <strong>Configure JIRA issue mappings</strong><br />
            For each event type:
            <ul class="list-disc list-inside ml-4 mt-1 space-y-1">
              <li>
                Type in the JIRA Issue field to see autocomplete suggestions
              </li>
              <li>
                Select from dropdown or enter issue key manually (e.g., ADM-6)
              </li>
              <li>
                Use team presets (Sally/Pangyo) to auto-fill multiple mappings
              </li>
              <li>Review dates and hours for each event type</li>
            </ul>
          </li>
          <li>
            <strong>Save and create work logs</strong><br />
            Click "Save" to store your mappings for future auto-matching and
            automatically create work log entries in JIRA for all configured
            events.
            <br /><strong>Note:</strong> You need to upload the .ics file each
            time, but your issue key mappings are saved for future sessions.
          </li>
        </ol>

        <div class="modal-action mt-4">
          <button class="btn" @click="showCalInfo = false">Close</button>
        </div>
      </div>
      <form
        method="dialog"
        class="modal-backdrop"
        @submit.prevent="showCalInfo = false"
      >
        <button>close</button>
      </form>
    </dialog>
  </div>
</template>

<script setup lang="ts">
// @ts-nocheck
import CalendarMonth from "~/components/CalendarMonth.vue";
import WorklogModal from "~/components/WorklogModal.vue";
import CeremonyConfigModal from "~/components/CeremonyConfigModal.vue";
import { ref, computed } from "vue";
import { jiraFetch } from "~/composables/useJiraApi";
import { useWorklogStore } from "~/composables/useWorklogStore";
import { useToastStore } from "~/composables/useToastStore";
import dayjs from "dayjs";
import utc from "dayjs/plugin/utc";
import timezone from "dayjs/plugin/timezone";
import isoWeek from "dayjs/plugin/isoWeek";
import isSameOrBefore from "dayjs/plugin/isSameOrBefore";
import { useJiraCredentials } from "~/composables/useJiraCredentials";
import { useCalendarLoading } from "~/composables/useCalendarLoading";
import { useCeremonyConfig } from "~/composables/useCeremonyConfig";
import ICAL from "ical.js";
import { setIcsContent, clearIcsContent } from "~/composables/useIcsStorage";

// Extend dayjs with required plugins once
dayjs.extend(utc);
dayjs.extend(timezone);
dayjs.extend(isoWeek);
dayjs.extend(isSameOrBefore);

// Set Thailand timezone as default
dayjs.tz.setDefault("Asia/Bangkok");

const selectedDate = ref(null);
const showWorklog = ref(false);
const autoFilling = ref(false);
const showCalInfo = ref(false);
const showConfigModal = ref(false);
const calRef = ref(null);
const fileInput = ref(null);

// Access global work-log store helpers
const { getLogs, fetchMonth, addHours, setLogs, markUpdated, getHours } =
  useWorklogStore();
const { addToast } = useToastStore();
const {
  configs: ceremonyConfigs,
  eventData,
  // setConfigs,
  setEventData,
} = useCeremonyConfig();

// Check if Jira credentials are present
const { email, token } = useJiraCredentials();
const hasCreds = computed(() => {
  // When rendering on the server we cannot access localStorage, so we assume
  // missing credentials to avoid prematurely enabling privileged controls.
  // The computed will re-evaluate on the client once hydrated.
  if (import.meta.server) return false;
  return !!email.value && !!token.value;
});

// Note: hasStoredIcs removed - no longer using persistent storage

// Shared calendar loading flag
const { loading: calendarLoading } = useCalendarLoading();

function onDaySelected(date) {
  if (!hasCreds.value) return;
  selectedDate.value = date;
  showWorklog.value = true;
}

// ---------------- Auto-Fill Ceremonies -------------------------------------
function confirmAutoFill() {
  if (autoFilling.value || !hasCreds.value) return;
  const anchorDate = calRef.value?.getAnchor?.() ?? new Date();
  const monthLabel = dayjs(anchorDate).format("MMM YYYY");
  if (
    confirm(
      `This will automatically add ceremony worklogs for ${monthLabel}. This is the legacy method. Continue?`
    )
  ) {
    autoFillCeremonies();
  }
}

async function autoFillCeremonies() {
  autoFilling.value = true;
  const anchorDate = calRef.value?.getAnchor?.() ?? new Date();
  const monthStart = dayjs(anchorDate).startOf("month");
  const monthEnd = dayjs(anchorDate).endOf("month");

  try {
    // Ensure we have up-to-date cache before checking duplicates
    await fetchMonth(monthStart.toDate());

    const creations = [];
    const hoursByDay = {};
    const newLogsByDate = {};

    for (
      let d = monthStart.clone();
      d.isSameOrBefore(monthEnd, "day");
      d = d.add(1, "day")
    ) {
      const weekday = d.day(); // 0 = Sun .. 6 = Sat
      if (weekday === 0 || weekday === 6) continue; // skip weekends

      const tasks = ceremonyTasksForDate(d);
      if (tasks.length === 0) continue;

      const existing = getLogs(d.toDate());

      // Filter out duplicates first
      const toAdd = tasks.filter(
        (t) =>
          !existing.some(
            (l) =>
              l.issueKey === t.issueKey &&
              l.timeSpentSeconds === Math.round(t.hours * 3600)
          )
      );

      if (toAdd.length === 0) continue;

      const existingHours = existing.reduce(
        (s, l) => s + (l.timeSpentSeconds ?? 0) / 3600,
        0
      );
      const addHours = toAdd.reduce((s, t) => s + t.hours, 0);

      if (existingHours + addHours > 8) {
        // exceed 8h cap – skip whole day
        continue;
      }

      // Queue creations and track hours tally
      toAdd.forEach((t) => {
        creations.push({ date: d.toDate(), ...t });
        // Prepare container for merging later
        const keyLogs = dayjs(d).format("YYYY-MM-DD");
        if (!newLogsByDate[keyLogs]) newLogsByDate[keyLogs] = [];
        hoursByDay[keyLogs] = (hoursByDay[keyLogs] || 0) + t.hours;
      });
    }

    // Process creations in parallel batches of 20
    const batchSize = 20;
    for (let i = 0; i < creations.length; i += batchSize) {
      const batch = creations.slice(i, i + batchSize);
      const results = await Promise.all(
        batch.map((item) => createWorklog(item.issueKey, item.hours, item.date))
      );
      results.forEach((resp, idx) => {
        const c = batch[idx];
        const iso = dayjs(c.date).format("YYYY-MM-DD");
        newLogsByDate[iso].push({
          id: resp?.id ?? "auto-" + Date.now() + "-" + idx,
          issueKey: c.issueKey,
          summary: "",
          timeSpentSeconds: Math.round(c.hours * 3600),
        });
      });
    }

    // Update local hours tally for immediate calendar highlight
    Object.entries(hoursByDay).forEach(([iso, hrs]) => {
      const dateObj = new Date(`${iso}T00:00:00`);
      const prevHours = getHours(dateObj);
      addHours(dateObj, hrs);
      const newHours = getHours(dateObj);
      // Trigger recent-update highlight for calendar cell using prev and new hours
      markUpdated(dateObj, prevHours, newHours);
    });

    // Merge newly created logs into central store so WorklogModal shows them immediately
    Object.entries(newLogsByDate).forEach(([iso, items]) => {
      const dateObj = new Date(`${iso}T00:00:00`);
      const merged = [...getLogs(dateObj), ...items];
      setLogs(dateObj, merged);
    });

    // Refresh store to include newly created logs so modal + duplicate checks stay in sync
    await fetchMonth(monthStart.toDate());

    addToast(
      `Ceremony worklogs have been added for ${monthStart.format("MMM YYYY")}.`,
      "success"
    );
  } catch (err) {
    console.error(err);
    addToast(
      "Failed to auto-fill ceremonies. See console for details.",
      "error"
    );
  } finally {
    autoFilling.value = false;
  }
}

function ceremonyTasksForDate(d) {
  const tasks = [{ issueKey: "ADM-6", hours: 1 }]; // All days
  const weekEven = d.isoWeek() % 2 === 0;
  const weekday = d.day();

  if (weekEven) {
    if (weekday === 2 || weekday === 4 || weekday === 5) {
      tasks.push({ issueKey: "ADM-17", hours: 0.25 });
    }
  } else {
    if (weekday === 1) {
      tasks.push({ issueKey: "ADM-17", hours: 0.25 });
      tasks.push({ issueKey: "ADM-18", hours: 0.5 });
      tasks.push({ issueKey: "ADM-19", hours: 1 });
    } else if (weekday === 2 || weekday === 4) {
      tasks.push({ issueKey: "ADM-17", hours: 0.25 });
    } else if (weekday === 5) {
      tasks.push({ issueKey: "ADM-17", hours: 0.25 });
      tasks.push({ issueKey: "ADM-20", hours: 1 });
      tasks.push({ issueKey: "ADM-16", hours: 1 });
      tasks.push({ issueKey: "ADM-18", hours: 1 });
    }
  }
  return tasks;
}

function createWorklog(issueKey, hours, dateObj) {
  const startedStr = dayjs(dateObj)
    .tz("Asia/Bangkok")
    .hour(9)
    .minute(0)
    .second(0)
    .millisecond(0)
    .format("YYYY-MM-DDTHH:mm:ss.SSSZZ");

  const payload = {
    started: startedStr,
    timeSpentSeconds: Math.round(hours * 3600),
    comment: {
      type: "doc",
      version: 1,
      content: [{ type: "paragraph", content: [] }],
    },
  };

  return jiraFetch(`rest/api/3/issue/${issueKey}/worklog`, {
    method: "POST",
    body: JSON.stringify(payload),
  });
}

// ---------------- Auto-Fill From Calendar ----------------------------------
/**
 * Parses .ics file content to extract calendar events within the specified date range.
 * @param {Date} start - The start of the date range.
 * @param {Date} end - The end of the date range.
 * @param {string} icsContent - The .ics file content to parse.
 * @returns {Promise<Array<{summary: string, start: {dateTime: string}, end: {dateTime: string}}>>} A promise that resolves to an array of calendar events.
 */
async function fetchCalendarEvents(start, end, icsContent = null) {
  if (!icsContent) {
    console.log("No .ics content provided");
    return [];
  }

  try {
    // Parse the .ics content
    const jcalData = ICAL.parse(icsContent);
    const comp = new ICAL.Component(jcalData);

    // Get all VEVENT components
    const vevents = comp.getAllSubcomponents("vevent");

    const events = [];
    const startDate = dayjs(start);
    const endDate = dayjs(end);

    for (const vevent of vevents) {
      const event = new ICAL.Event(vevent);

      // Get event summary (title)
      const summary = event.summary || "Untitled Event";

      // Handle recurring events by expanding occurrences within range
      if (event.isRecurring()) {
        const iterator = event.iterator();
        let next = iterator.next();

        while (next) {
          const occStartDay = dayjs(next.toJSDate());

          // Stop iterating once we are past the desired end range
          if (occStartDay.isAfter(endDate, "day")) break;

          // Skip occurrences before the start range
          if (occStartDay.isBefore(startDate, "day")) {
            next = iterator.next();
            continue;
          }

          // Get full occurrence details (accounts for EXDATE, DTSTART/DTEND, etc.)
          const { startDate: occStart, endDate: occEnd } =
            event.getOccurrenceDetails(next);

          events.push({
            summary,
            start: { dateTime: occStart.toJSDate().toISOString() },
            end: { dateTime: occEnd.toJSDate().toISOString() },
          });

          next = iterator.next();
        }
        continue; // Skip standard processing for recurring master event
      }

      // Get start and end times for non-recurring (single) events
      const startTime = event.startDate;
      const endTime = event.endDate;

      if (!startTime || !endTime) continue;

      // Convert ICAL time to JavaScript Date
      const startJs = startTime.toJSDate();
      const endJs = endTime.toJSDate();

      // Check if event falls within the specified month range
      const eventStart = dayjs(startJs);
      const eventEnd = dayjs(endJs);

      // Include event if it overlaps with the target month
      if (
        eventStart.isBefore(endDate, "day") &&
        eventEnd.isAfter(startDate, "day")
      ) {
        events.push({
          summary: summary,
          start: {
            dateTime: startJs.toISOString(),
          },
          end: {
            dateTime: endJs.toISOString(),
          },
        });
      }
    }

    console.log(
      `Parsed ${events.length} events from .ics file for ${startDate.format(
        "MMM YYYY"
      )}`
    );

    return events;
  } catch (error) {
    console.error("Error parsing .ics file:", error);
    throw new Error(
      "Failed to parse .ics file. Please ensure it is a valid calendar file."
    );
  }
}

function confirmAutoFillFromCalendar() {
  if (autoFilling.value || !hasCreds.value) return;
  const anchorDate = calRef.value?.getAnchor?.() ?? new Date();
  const monthLabel = dayjs(anchorDate).format("MMM YYYY");

  // Always require file upload - no persistent storage
  if (
    confirm(
      `This will parse calendar events from an .ics file for ${monthLabel} and open the configuration with auto-filled event titles. Please select an .ics file to continue.`
    )
  ) {
    // Clear any existing session data and trigger file input for auto-fill flow
    clearIcsContent();
    isAutoFillFlow.value = true;
    fileInput.value?.click();
  }
}

// Track if we're in auto-fill flow to open configure modal after file selection
const isAutoFillFlow = ref(false);

// Function to extract event information and open configure modal
async function openConfigureWithAutoFilledTitles(icsContent) {
  const anchorDate = calRef.value?.getAnchor?.() ?? new Date();
  const monthStart = dayjs(anchorDate).startOf("month");
  const monthEnd = dayjs(anchorDate).endOf("month");

  try {
    // Parse calendar events to extract information
    const calendarEvents = await fetchCalendarEvents(
      monthStart.toDate(),
      monthEnd.toDate(),
      icsContent
    );

    if (calendarEvents.length === 0) {
      addToast("No events found in the selected calendar file.", "warn");
      return;
    }

    // Group events by title and collect dates/hours
    const eventMap = new Map();

    calendarEvents.forEach((event) => {
      const title = event.summary?.trim() || "Untitled Event";
      const startTime = dayjs(event.start.dateTime);
      const endTime = dayjs(event.end.dateTime);
      const hours = Math.round(endTime.diff(startTime, "minute") / 15) * 0.25; // Round to nearest 0.25 hour

      if (!eventMap.has(title)) {
        eventMap.set(
          title,
          reactive({
            title,
            issueKey: "", // Will be filled by user or preset
            dates: [],
            // Initialize search-related reactive properties
            suggestions: [],
            searching: false,
            issueSummary: null,
            issueType: null,
            issueError: null,
          })
        );
      }

      eventMap.get(title).dates.push({
        date: startTime.format("YYYY-MM-DD"),
        hours: Math.max(0.25, hours), // Minimum 0.25 hours
      });
    });

    // Convert map to array
    const uniqueEvents = Array.from(eventMap.values());

    // Auto-fill issue keys from saved ceremony-mappings
    let matchedCount = 0;
    if (ceremonyConfigs.value && ceremonyConfigs.value.length > 0) {
      uniqueEvents.forEach((event) => {
        // Find matching config by checking if event title contains config title or vice versa
        const matchingConfig = ceremonyConfigs.value.find(
          (config) =>
            event.title.toLowerCase().includes(config.title.toLowerCase()) ||
            config.title.toLowerCase().includes(event.title.toLowerCase())
        );

        if (matchingConfig && matchingConfig.issueKey) {
          event.issueKey = matchingConfig.issueKey;
          matchedCount++;
        }
      });
    }

    // Set the event data and open the modal
    setEventData(uniqueEvents);
    showConfigModal.value = true;

    if (matchedCount > 0) {
      addToast(
        `Found ${uniqueEvents.length} unique event types (${matchedCount} auto-filled from saved mappings). Review and adjust as needed.`,
        "success"
      );
    } else {
      addToast(
        `Found ${uniqueEvents.length} unique event types with ${calendarEvents.length} total events. Configure issue key mappings and select a team preset.`,
        "info"
      );
    }
  } catch (error) {
    console.error("Error parsing calendar events:", error);
    addToast(
      "Failed to parse calendar events. Please check your .ics file.",
      "error"
    );
  }
}

// Handle when config modal is saved - trigger auto-fill
async function onConfigSaved() {
  autoFilling.value = true;

  try {
    // Get event data with configured issue keys
    const validEvents = eventData.value.filter((event) => event.issueKey);

    if (validEvents.length === 0) {
      addToast(
        "No valid event mappings found. Please configure issue keys.",
        "warn"
      );
      return;
    }

    await fetchMonth(new Date()); // Ensure month data is loaded

    const creations = [];
    const hoursByDay = {};
    const newLogsByDate = {};

    for (const event of validEvents) {
      for (const dateEntry of event.dates) {
        const workDate = new Date(dateEntry.date + "T00:00:00");
        const existing = getLogs(workDate);

        // Check for duplicates
        const isDuplicate = existing.some(
          (l) =>
            l.issueKey === event.issueKey &&
            l.timeSpentSeconds === Math.round(dateEntry.hours * 3600)
        );

        if (isDuplicate) {
          console.log(
            `Skipping duplicate: ${event.title} on ${dateEntry.date}`
          );
          continue;
        }

        // Check 8-hour daily limit
        const existingHours = existing.reduce(
          (s, l) => s + (l.timeSpentSeconds ?? 0) / 3600,
          0
        );

        if (existingHours + dateEntry.hours > 8) {
          console.log(
            `Skipping ${event.title} on ${dateEntry.date} - would exceed 8h limit`
          );
          continue;
        }

        // Queue for creation
        creations.push({
          date: workDate,
          issueKey: event.issueKey,
          hours: dateEntry.hours,
          title: event.title,
        });

        // Track for local storage update
        const dateKey = dayjs(workDate).format("YYYY-MM-DD");
        if (!newLogsByDate[dateKey]) newLogsByDate[dateKey] = [];
        hoursByDay[dateKey] = (hoursByDay[dateKey] || 0) + dateEntry.hours;
      }
    }

    if (creations.length === 0) {
      addToast(
        "No new work logs to create (all would be duplicates or exceed limits).",
        "info"
      );
      return;
    }

    addToast(`Creating ${creations.length} work log entries...`, "info");

    // Process creations in parallel batches of 20
    const batchSize = 20;
    for (let i = 0; i < creations.length; i += batchSize) {
      const batch = creations.slice(i, i + batchSize);
      const results = await Promise.all(
        batch.map((item) => createWorklog(item.issueKey, item.hours, item.date))
      );

      results.forEach((resp, idx) => {
        const c = batch[idx];
        const dateKey = dayjs(c.date).format("YYYY-MM-DD");
        newLogsByDate[dateKey].push({
          id: resp?.id ?? "auto-" + Date.now() + "-" + idx,
          issueKey: c.issueKey,
          summary: c.title,
          timeSpentSeconds: Math.round(c.hours * 3600),
        });
      });
    }

    // Update local storage
    Object.entries(hoursByDay).forEach(([dateKey, hrs]) => {
      const dateObj = new Date(`${dateKey}T00:00:00`);
      const prevHours = getHours(dateObj);
      addHours(dateObj, hrs);
      const newHours = getHours(dateObj);
      markUpdated(dateObj, prevHours, newHours);
    });

    // Merge newly created logs into central store
    Object.entries(newLogsByDate).forEach(([dateKey, items]) => {
      const dateObj = new Date(`${dateKey}T00:00:00`);
      const merged = [...getLogs(dateObj), ...items];
      setLogs(dateObj, merged);
    });

    await fetchMonth(new Date()); // Refresh to show updated data
    addToast(
      `Successfully logged ${creations.length} work entries from calendar!`,
      "success"
    );
  } catch (error) {
    console.error("Error during auto-fill:", error);
    addToast(
      "Failed to auto-fill work logs. See console for details.",
      "error"
    );
  } finally {
    autoFilling.value = false;
  }
}

async function handleFileSelect(event) {
  const file = event.target.files?.[0];
  if (!file) return;

  if (!file.name.endsWith(".ics")) {
    addToast("Please select a valid .ics file.", "error");
    return;
  }

  try {
    const content = await readFileAsText(file);
    // Store for current session only
    setIcsContent(String(content));

    if (isAutoFillFlow.value) {
      // Extract event titles and open configure modal
      isAutoFillFlow.value = false;
      await openConfigureWithAutoFilledTitles(String(content));
    } else {
      // Direct auto-fill (legacy flow, if needed)
      await autoFillFromCalendar(String(content));
    }
  } catch (err) {
    console.error("Error reading file:", err);
    addToast("Failed to read the .ics file.", "error");
    isAutoFillFlow.value = false;
  } finally {
    // Reset file input
    if (fileInput.value) {
      fileInput.value.value = "";
    }
  }
}

function readFileAsText(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = (e) => resolve(e.target?.result);
    reader.onerror = reject;
    reader.readAsText(file);
  });
}

async function autoFillFromCalendar(icsContent = null) {
  autoFilling.value = true;
  const anchorDate = calRef.value?.getAnchor?.() ?? new Date();
  const monthStart = dayjs(anchorDate).startOf("month");
  const monthEnd = dayjs(anchorDate).endOf("month");

  try {
    await fetchMonth(monthStart.toDate());
    const calendarEvents = await fetchCalendarEvents(
      monthStart.toDate(),
      monthEnd.toDate(),
      icsContent
    );

    const creations = [];
    const hoursByDay = {};
    const newLogsByDate = {};

    for (const event of calendarEvents) {
      if (!event.summary || !event.start?.dateTime || !event.end?.dateTime)
        continue;

      const eventTitle = event.summary.toLowerCase();
      const eventDate = dayjs(event.start.dateTime);

      const matchingConfig = ceremonyConfigs.value.find((c) =>
        eventTitle.includes(c.title.toLowerCase())
      );

      if (!matchingConfig) continue;

      const d = eventDate.startOf("day");
      const existing = getLogs(d.toDate());

      const startTime = dayjs(event.start.dateTime);
      const endTime = dayjs(event.end.dateTime);
      // Round to nearest 0.25 hour
      const hours = Math.round(endTime.diff(startTime, "minute") / 15) * 0.25;

      if (hours <= 0) continue;

      const toAdd = {
        issueKey: matchingConfig.issueKey,
        hours,
      };

      // Filter out duplicates
      const isDuplicate = existing.some(
        (l) =>
          l.issueKey === toAdd.issueKey &&
          l.timeSpentSeconds === Math.round(toAdd.hours * 3600)
      );
      if (isDuplicate) continue;

      const existingHours = existing.reduce(
        (s, l) => s + (l.timeSpentSeconds ?? 0) / 3600,
        0
      );
      if (existingHours + toAdd.hours > 8) {
        continue; // Skip day if it exceeds 8h cap
      }

      creations.push({ date: d.toDate(), ...toAdd });
      const keyLogs = d.format("YYYY-MM-DD");
      if (!newLogsByDate[keyLogs]) newLogsByDate[keyLogs] = [];
      hoursByDay[keyLogs] = (hoursByDay[keyLogs] || 0) + toAdd.hours;
    }

    // Process creations in parallel batches of 20
    const batchSize = 20;
    for (let i = 0; i < creations.length; i += batchSize) {
      const batch = creations.slice(i, i + batchSize);
      const results = await Promise.all(
        batch.map((item) => createWorklog(item.issueKey, item.hours, item.date))
      );
      results.forEach((resp, idx) => {
        const c = batch[idx];
        const iso = dayjs(c.date).format("YYYY-MM-DD");
        newLogsByDate[iso].push({
          id: resp?.id ?? "auto-" + Date.now() + "-" + idx,
          issueKey: c.issueKey,
          summary: "",
          timeSpentSeconds: Math.round(c.hours * 3600),
        });
      });
    }

    // Update local hours tally for immediate calendar highlight
    Object.entries(hoursByDay).forEach(([iso, hrs]) => {
      const dateObj = new Date(`${iso}T00:00:00`);
      const prevHours = getHours(dateObj);
      addHours(dateObj, hrs);
      const newHours = getHours(dateObj);
      markUpdated(dateObj, prevHours, newHours);
    });

    // Merge newly created logs into central store so WorklogModal shows them immediately
    Object.entries(newLogsByDate).forEach(([iso, items]) => {
      const dateObj = new Date(`${iso}T00:00:00`);
      const merged = [...getLogs(dateObj), ...items];
      setLogs(dateObj, merged);
    });

    await fetchMonth(monthStart.toDate());

    addToast(
      `Worklogs from calendar events have been added for ${monthStart.format(
        "MMM YYYY"
      )}.`,
      "success"
    );
  } catch (err) {
    console.error(err);
    addToast(
      "Failed to auto-fill from calendar. See console for details.",
      "error"
    );
  } finally {
    autoFilling.value = false;
  }
}
</script>
