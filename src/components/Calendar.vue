<script setup>
import { ref, onMounted } from 'vue';
import FullCalendar from '@fullcalendar/vue3';
import dayGridPlugin from '@fullcalendar/daygrid';
import interactionPlugin from '@fullcalendar/interaction';
import { DynamoDBClient, ScanCommand } from '@aws-sdk/client-dynamodb';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import EventPopup from './EventPopup.vue'; // Import EventPopup component
import DocumentEditor from './DocumentEditor.vue';

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
const selectedFile = ref(null);  // To store the fetched DOCX file
const selectedDocumentId = ref(null); // To store document Id

// Initialize the S3 client
const s3Client = new S3Client({
  region: region,
  credentials: {
    accessKeyId: accessKeyId,
    secretAccessKey: secretAccessKey,
  }
});

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
            docxLink: item.s3link?.S, // Assuming docxLink contains the link to the file
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

const openDocumentEditor = async (docxLink) => {
  try {
    selectedDocumentId.value = 'document.docx'
    
    // Extract bucket and key from the docxLink (assuming the link is an S3 URL)
    const url = new URL(docxLink);
    console.log('Hostname:', url.hostname);
    const bucketName = url.pathname.split('/')[1];
    const fileKey = url.pathname.split('/').slice(2).join('/');

    console.log('Bucket Name:', bucketName);
    console.log('File Key:', fileKey);

    // Create the GetObject command
    const command = new GetObjectCommand({
      Bucket: bucketName,
      Key: fileKey,
    });

    // Send the command to S3 to get the file
    const data = await s3Client.send(command);

    // The file's content is in the Body (this is a stream in Node.js or a ReadableStream in the browser)
    const fileStream = data.Body;

    // If it's a ReadableStream (in the browser)
    if (fileStream instanceof ReadableStream) {
      const fileBlob = await streamToBlob(fileStream);  // Convert the ReadableStream to a Blob

      // Create a new File object from the Blob (with a filename and MIME type)
      const file = new File([fileBlob], 'document.docx', { 
        type: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document' 
      });

      // Save the File object in selectedFile
      selectedFile.value = file;
      console.log('File successfully downloaded and saved as a File object:', file);
    } else {
      // Handle case where Body is a Node.js stream (if this is being run in Node.js)
      const fileBlob = await streamToBlob(fileStream);
      
      // Create a new File object from the Blob (with a filename and MIME type)
      const file = new File([fileBlob], 'document.docx', { 
        type: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document' 
      });

      // Save the File object in selectedFile
      selectedFile.value = file;
      console.log('File successfully downloaded and saved as a File object:', file);
    }
  } catch (error) {
    console.error('Error downloading the DOCX file from S3:', error);
  }
};

// Helper function to convert a stream to a Blob (for browser environment)
const streamToBlob = (stream) => {
  return new Promise((resolve, reject) => {
    const reader = stream.getReader();
    const chunks = [];

    reader.read().then(function processChunk({ done, value }) {
      if (done) {
        resolve(new Blob(chunks));
        return;
      }

      chunks.push(value);
      reader.read().then(processChunk);
    }).catch(reject);
  });
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
      @open-document-editor="openDocumentEditor"
      @close="closePopup"
    />

    <DocumentEditor 
      v-if="selectedFile" 
      :documentId="selectedDocumentId" 
      :initialData="selectedFile" 
      :readOnly="true"
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
