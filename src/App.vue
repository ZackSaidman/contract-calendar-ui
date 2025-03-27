<script setup>
import Calendar from './components/Calendar.vue';
import DocxUploader from './components/DocxUploader.vue';
</script>

<template>
  <main>
    <Calendar />
    <DocxUploader />
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
