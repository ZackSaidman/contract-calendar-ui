<script setup>
import { ref, onMounted } from 'vue';
import FullCalendar from '@fullcalendar/vue3';
import dayGridPlugin from '@fullcalendar/daygrid';
import interactionPlugin from '@fullcalendar/interaction';
import { DynamoDBClient, ScanCommand } from '@aws-sdk/client-dynamodb';

// Access your AWS credentials and region from environment variables
const region = import.meta.env.VITE_AWS_REGION;
const accessKeyId = import.meta.env.VITE_AWS_ACCESS_KEY_ID;
const secretAccessKey = import.meta.env.VITE_AWS_SECRET_ACCESS_KEY;

// Create a reference for the events
const events = ref([]);

// Initialize DynamoDB Client with direct credentials from environment variables
const dynamoClient = new DynamoDBClient({
  region,
  credentials: {
    accessKeyId,
    secretAccessKey,
  },
});

// Handle date click event
const handleDateClick = (arg) => {
  alert('date click! ' + arg.dateStr);
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
        const tableData = item.tableData?.L || []; // Safely access tableData if it exists
        tableData.forEach(event => {
          // Push each event object to the events array
          console.log(event)
          events.value.push({
            title: event.M.title?.S,
            date: event.M.date?.S
          });
        });
      });
      console.log(events)
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
  <FullCalendar :options="calendarOptions" />
</template>
