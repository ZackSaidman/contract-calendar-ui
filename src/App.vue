<script setup>
import Calendar from './components/Calendar.vue';
import DocxUploader from './components/DocxUploader.vue';
</script>

<template>
  <main>
    <DocxUploader />
    <Calendar />
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
