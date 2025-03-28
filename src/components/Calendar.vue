<script setup>
import FullCalendar from '@fullcalendar/vue3';
import dayGridPlugin from '@fullcalendar/daygrid';
import interactionPlugin from '@fullcalendar/interaction';

// Handle date click event
const handleDateClick = (arg) => {
  alert('date click! ' + arg.dateStr);
};

// Calendar options
const calendarOptions = {
  contentHeight: 'auto',
  plugins: [dayGridPlugin, interactionPlugin],
  initialView: 'dayGridMonth',
  dateClick: handleDateClick, // Pass the function reference
  events: [
    { title: 'event 1', date: '2025-03-01' },
    { title: 'event 2', date: '2025-03-02' }
  ]
};
</script>

<template>
  <FullCalendar :options="calendarOptions" />
</template>
