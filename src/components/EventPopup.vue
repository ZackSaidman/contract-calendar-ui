<script setup>
import { ref, defineProps, defineEmits, watch, onUnmounted, nextTick } from 'vue';

// Define props to receive the event data and visibility flag
const props = defineProps({
  selectedEvent: Object,
  isVisible: Boolean,
});

// Define emit event to close the popup and to open the document editor
const emit = defineEmits(['close', 'open-document-editor']);

const popupRef = ref(null);
let eventListenerActive = false; // Prevent premature event firing

// Close the popup
const closePopup = () => {
  emit('close');
};

// Emit the event to open the document editor
const openDocumentEditor = () => {
  emit('open-document-editor', props.selectedEvent.docxLink); // Pass the DOCX link
};

// Click outside handler
const handleClickOutside = (event) => {
  if (eventListenerActive && props.isVisible && popupRef.value && !popupRef.value.contains(event.target)) {
    closePopup();
  }
};

// Attach event listener after popup opens
watch(() => props.isVisible, async (newVal) => {
  if (newVal) {
    await nextTick(); // Wait for DOM update
    setTimeout(() => {
      eventListenerActive = true;
      document.addEventListener('click', handleClickOutside);
    }, 100);
  } else {
    document.removeEventListener('click', handleClickOutside);
  }
});

// Cleanup on unmount
onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside);
});
</script>

<template>
  <div v-if="props.isVisible" class="event-popup">
    <div class="popup-content" ref="popupRef">
      <h3>{{ props.selectedEvent?.title }}</h3>
      <p><strong>Date:</strong> {{ props.selectedEvent?.date }}</p>
      <p><strong>Doc Text:</strong> "{{ props.selectedEvent?.text }}"</p>
      <p><strong>Link to DOCX:</strong> 
        <a href="#" @click.prevent="openDocumentEditor">Open DOCX</a>
      </p>
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
