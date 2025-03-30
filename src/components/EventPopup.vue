<script setup>
import { defineProps, defineEmits } from 'vue';

// Define props to receive the event data and visibility flag
const props = defineProps({
  selectedEvent: Object,
  isVisible: Boolean,
});

// Define emit event to close the popup
const emit = defineEmits(['close']);

// Close the popup
const closePopup = () => {
  emit('close');
};
</script>

<template>
  <div v-if="props.isVisible" class="event-popup">
    <div class="popup-content">
      <h3>{{ props.selectedEvent.title }}</h3>
      <p><strong>Date:</strong> {{ props.selectedEvent.date }}</p>
      <p><strong>Link to DOCX:</strong> <a :href="props.selectedEvent.docxLink" target="_blank">Download DOCX</a></p>
      <button @click="closePopup">Close</button>
    </div>
  </div>
</template>

<style scoped>
.event-popup {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 20px;
  background-color: white;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  border-radius: 8px;
  z-index: 1000;
  width: 300px;
  max-width: 90%;
}

.popup-content {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.popup-content h3 {
  margin: 0;
  font-size: 18px;
  font-weight: bold;
}

.popup-content p {
  margin: 5px 0;
}

button {
  margin-top: 10px;
  padding: 8px 16px;
  background-color: #42b983;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:hover {
  background-color: #36976d;
}
</style>
