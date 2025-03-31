<script setup>
import { defineProps, ref, watch, onMounted, onUnmounted } from 'vue';
import FullCalendar from '@fullcalendar/vue3';
import dayGridPlugin from '@fullcalendar/daygrid';
import interactionPlugin from '@fullcalendar/interaction';
import { DynamoDBClient, ScanCommand } from '@aws-sdk/client-dynamodb';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import EventPopup from './EventPopup.vue'; // Import EventPopup component
import DocumentEditor from './DocumentEditor.vue';
import Modal from './Modal.vue';

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
const isModalVisible = ref(false); // Controls whether the modal is visible
const popupRef = ref(null); // Reference to the popup

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

// Prop for triggering refresh from parent component (App.vue)
const props = defineProps({
  refresh: Boolean, // This will be passed from App.vue to trigger the refresh
});

// Watch for the 'refresh' prop to trigger the fetch of events
watch(() => props.refresh, (newVal) => {
  if (newVal) {
    fetchEventsFromDynamoDB(); // Refresh the events when 'refresh' is true
  }
});

// Fetch events from DynamoDB
const fetchEventsFromDynamoDB = async () => {
  try {
    const newEvents = [];

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
          newEvents.push({
            title: item.filename?.S,
            date: event.M.date?.S,
            text: event.M.text?.S,
            docxLink: item.s3link?.S, // Assuming docxLink contains the link to the file
          });
        });
      });

      const existingEventKeys = new Set(events.value.map(e => `${e.title}-${e.date}`));

      const uniqueNewEvents = newEvents.filter(e => !existingEventKeys.has(`${e.title}-${e.date}`));

      if (uniqueNewEvents.length > 0) {

        // Efficiently add only unique events
        events.value.push(...uniqueNewEvents);
      }
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
    const bucketName = url.pathname.split('/')[1];
    const fileKey = url.pathname.split('/').slice(2).join('/');

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
      const file = new File([fileBlob], selectedDocumentId.value, { 
        type: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document' 
      });

      // Save the File object in selectedFile
      selectedFile.value = file;
    } else {
      // Handle case where Body is a Node.js stream (if this is being run in Node.js)
      const fileBlob = await streamToBlob(fileStream);
      
      // Create a new File object from the Blob (with a filename and MIME type)
      const file = new File([fileBlob], selectedDocumentId.value, { 
        type: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document' 
      });

      // Save the File object in selectedFile
      selectedFile.value = file;
    }

    // Show the modal once the file is ready
    isModalVisible.value = true;
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
  document.addEventListener('click', handleClickOutside);
  fetchEventsFromDynamoDB();
});

onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside);
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

// Function to close popup when clicking outside
const handleClickOutside = (event) => {
  if (popupVisible.value && popupRef.value && !popupRef.value.contains(event.target)) {
    closePopup();
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

// Close the modal
const closeModal = () => {
  console.log('Closing Modal');
  isModalVisible.value = false;
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
      @close="popupVisible = false"
    />

    <!-- Modal with DocumentEditor inside -->
    <Modal :isVisible="isModalVisible" @close="closeModal">
      <DocumentEditor 
        v-if="selectedFile" 
        :documentId="selectedDocumentId" 
        :initialData="selectedFile" 
        :readOnly="true"
      />
    </Modal>
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
