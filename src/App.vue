<script setup>
import { ref } from 'vue';
import Calendar from './components/Calendar.vue';
import DocxUploader from './components/DocxUploader.vue';

// Define a ref to track if the events need to be refreshed
const refreshCalendar = ref(false);

// Method to handle file upload completion
const handleFileUploaded = () => {
  refreshCalendar.value = true; // Trigger a refresh in the calendar
  setTimeout(() => {
    refreshCalendar.value = false; // Reset after the refresh
  }, 100); // Reset the refresh flag after a brief delay
};
</script>

<template>
  <main>
    <DocxUploader @fileUploaded="handleFileUploaded" />
    <Calendar :refresh="refreshCalendar" />
    <br>
    <a href="https://www.flaticon.com/free-icons/rowing" title="rowing icons">Rowing icons created by Freepik - Flaticon</a>
  </main>
</template>

<style scoped>
@media (min-width: 1024px) {
  body {
    display: block; /* Change from flex to block to prevent compression */
  }

  #app {
    display: flex;
    justify-content: center; /* Center the calendar */
    align-items: center;     /* Center vertically */
    flex-direction: column;  /* Ensure items stack vertically */
  }
}

.fc {
  width: 100%;
  max-width: 100%;
  min-height: 480px;
}
</style>
