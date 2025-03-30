<script setup>
import { ref, onMounted } from 'vue';
import FullCalendar from '@fullcalendar/vue3';
import dayGridPlugin from '@fullcalendar/daygrid';
import interactionPlugin from '@fullcalendar/interaction';
import { DynamoDBClient, ScanCommand } from '@aws-sdk/client-dynamodb';
import EventPopup from './EventPopup.vue'; // Import EventPopup component

// AWS Config
const region = import.meta.env.VITE_AWS_REGION;
const accessKeyId = import.meta.env.VITE_AWS_ACCESS_KEY_ID;
const secretAccessKey = import.meta.env.VITE_AWS_SECRET_ACCESS_KEY;

// Create a reference for the events
const events = ref([]);
const selectedEvents = ref([]); // Stores events for the clicked date
const selectedDate = ref(null); // Stores the selected date
const selectedEvent = ref(null); // Stores the selected event for the popup
const popupVisible = ref(false); // Control visibility of the popup

// Initialize DynamoDB Client
const dynamoClient = new DynamoDBClient({
  region,
  credentials: {
    accessKeyId,
    secretAccessKey,
  },
});

// Fetch events from DynamoDB
const fetchEventsFromDynamoDB = async () => {
  try {
    const command = new ScanCommand({
      TableName: import.meta.env.VITE_DYNAMO_TABLE_NAME,
    });

    const data = await dynamoClient.send(command);

    // Check if Items are returned and safely map the attributes
    if (data.Items) {
      data.Items.forEach(item => {
        const tableData = item.tableData?.L || [];
        tableData.forEach(event => {
          // Push each event object to the events array
          events.value.push({
            title: event.M.title?.S,
            date: event.M.date?.S,
            docxLink: event.M.s3link?.S, // Assuming docxLink contains the link to the file
          });
        });
      });
    } else {
      console.error('No items found in DynamoDB response');
    }
  } catch (error) {
    console.error('Error fetching events from DynamoDB:', error);
  }
};

// Fetch events on component mount
onMounted(() => {
  fetchEventsFromDynamoDB();
});

// Handle date click event
const handleDateClick = (arg) => {
  selectedDate.value = arg.dateStr; // Store selected date
  selectedEvents.value = events.value.filter(event => event.date === arg.dateStr);
};

// Handle event click to display the event details in a popup
const handleEventClick = (info) => {
  const event = events.value.find(e => e.title === info.event.title && e.date === info.event.startStr);
  if (event) {
    selectedEvent.value = event;
    popupVisible.value = true; // Show the popup
  }
};

// Calendar options
const calendarOptions = {
  contentHeight: 'auto',
  plugins: [dayGridPlugin, interactionPlugin],
  initialView: 'dayGridMonth',
  events: events.value, // Dynamically load events from DynamoDB
  dateClick: handleDateClick, // Keep the date click functionality
  eventClick: handleEventClick, // Handle event click to show details in the popup
};

// Handle closing the popup
const closePopup = () => {
  popupVisible.value = false;
};
</script>

<template>
  <div>
    <FullCalendar :options="calendarOptions" />

    <!-- Event Box for selected date -->
    <div v-if="selectedDate" class="event-box">
      <h3>Events on {{ selectedDate }}</h3>
      <ul v-if="selectedEvents.length">
        <li v-for="(event, index) in selectedEvents" :key="index">
          {{ event.title }}
        </li>
      </ul>
      <p v-else>No events for this date.</p>
    </div>

    <!-- Event Popup -->
    <EventPopup
      :selectedEvent="selectedEvent"
      :isVisible="popupVisible"
      @close="closePopup"
    />
  </div>
</template>

<style scoped>
.event-box {
  margin-top: 20px;
  padding: 15px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background-color: #f9f9f9;
  box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.1);
}
</style>
