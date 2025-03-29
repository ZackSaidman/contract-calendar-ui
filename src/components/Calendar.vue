<script setup>
import { ref, onMounted } from 'vue';
import FullCalendar from '@fullcalendar/vue3';
import dayGridPlugin from '@fullcalendar/daygrid';
import interactionPlugin from '@fullcalendar/interaction';
import { DynamoDBClient, ScanCommand } from '@aws-sdk/client-dynamodb';

// AWS Config
const region = import.meta.env.VITE_AWS_REGION;
const accessKeyId = import.meta.env.VITE_AWS_ACCESS_KEY_ID;
const secretAccessKey = import.meta.env.VITE_AWS_SECRET_ACCESS_KEY;

// Create a reference for the events
const events = ref([]);
const selectedEvents = ref([]); // Stores events for the clicked date
const selectedDate = ref(null); // Stores the selected date

// Initialize DynamoDB Client
const dynamoClient = new DynamoDBClient({
  region,
  credentials: {
    accessKeyId,
    secretAccessKey,
  },
});

// Handle date click event
const handleDateClick = (arg) => {
  selectedDate.value = arg.dateStr; // Store selected date
  selectedEvents.value = events.value.filter(event => event.date === arg.dateStr);
};

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

// Calendar options
const calendarOptions = {
  contentHeight: 'auto',
  plugins: [dayGridPlugin, interactionPlugin],
  initialView: 'dayGridMonth',
  dateClick: handleDateClick, // Pass the function reference
  events: events.value, // Dynamically load events from DynamoDB
};
</script>

<template>
  <div>
    <FullCalendar :options="calendarOptions" />
    
    <!-- Event Box -->
    <div v-if="selectedDate" class="event-box">
      <h3>Events on {{ selectedDate }}</h3>
      <ul v-if="selectedEvents.length">
        <li v-for="(event, index) in selectedEvents" :key="index">
          {{ event.title }}
        </li>
      </ul>
      <p v-else>No events for this date.</p>
    </div>
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
