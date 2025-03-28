<template>
  <div class="upload-container">
    <label for="file-upload" class="upload-button" :class="{ 'is-loading': isUploading }">
      {{ isUploading ? 'Uploading...' : 'Upload DOCX' }}
    </label>
    <input id="file-upload" type="file" accept=".docx" @change="handleFileUpload" hidden />
  </div>
</template>

<script setup>
import { ref } from 'vue';
import AWS from 'aws-sdk';
import JSZip from 'jszip';
import { XMLParser } from 'fast-xml-parser';

// Uploading state for better UX
const isUploading = ref(false);

// AWS Configuration (use environment variables for security)
AWS.config.update({
  region: import.meta.env.VITE_AWS_REGION,
  accessKeyId: import.meta.env.VITE_AWS_ACCESS_KEY_ID,
  secretAccessKey: import.meta.env.VITE_AWS_SECRET_ACCESS_KEY,
});

const s3 = new AWS.S3();
const dynamoDB = new AWS.DynamoDB.DocumentClient();

const handleFileUpload = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  if (file.type !== 'application/vnd.openxmlformats-officedocument.wordprocessingml.document') {
    alert('Please upload a valid .docx file');
    return;
  }

  isUploading.value = true;

  try {
    // Read the .docx file using JSZip
    const zip = await JSZip.loadAsync(file);
    
    // Extract the document.xml content
    const docXml = await zip.file('word/document.xml').async('string');
    
    // Parse the XML content using fast-xml-parser
    const parser = new XMLParser();
    const result = parser.parse(docXml);

    // Log the entire parsed XML structure to debug
    console.log('Parsed XML structure:', result);

    // Extract the table data from the parsed XML (based on the structure of your DOCX file)
    const table = result['w:document']['w:body']['w:tbl'];

    if (!table) {
      alert('No table found in the document.');
      return;
    }

    // Initialize an array to store extracted table data
    const tableData = [];
    
    const rows = table['w:tr'];  // Rows are 'w:tr'

    // Log the rows to check the structure
    console.log('Table Rows:', rows);

    // Process rows to find the "Title" and "Date" headers in the first row
    let headersFound = false;
    
    rows.forEach((row, rowIndex) => {
      const cells = row['w:tc']; // Cells are 'w:tc'

      const rowData = cells.map((cell) => {
        const text = cell['w:p']?.['w:r']?.['w:t'];
        return text ? text.trim() : '';  // Return empty string if no text is found
      });

      // Log rowData for debugging
      console.log(`Row ${rowIndex}:`, rowData);

      // Check if this is the header row
      if (rowIndex === 0) {
        const [firstCell, secondCell] = rowData;

        // Ensure the first row contains "Title" and "Date"
        if (firstCell === 'Title' && secondCell === 'Date') {
          headersFound = true;
        } else {
          alert('Expected headers "Title" and "Date" not found in the first row.');
          return;
        }
      } else {
        // If it's not the header row, push the data into tableData
        if (rowData.length >= 2) {
          const [title, date] = rowData;
          tableData.push({ title, date });
        }
      }
    });

    if (!headersFound) {
      alert('Could not find the "Title" and "Date" headers.');
      return;
    }

    if (tableData.length === 0) {
      alert('No valid table data found.');
      return;
    }

    console.log('Extracted table data:', tableData);

    // Prepare file upload to S3
    const fileKey = `uploads/${Date.now()}-${file.name}`;

    // Upload to S3
    await s3.upload({
      Bucket: import.meta.env.VITE_S3_BUCKET_NAME,
      Key: fileKey,
      Body: file,
      ContentType: file.type,
      ACL: 'private', // Secure by default
    }).promise();
    console.log('File uploaded to S3:', fileKey);

    // Log to DynamoDB for each entry
    await dynamoDB.put({
      TableName: import.meta.env.VITE_DYNAMO_TABLE_NAME,
      Item: {
        filename: file.name,
        s3link: `https://s3.${import.meta.env.VITE_AWS_REGION}.amazonaws.com/${import.meta.env.VITE_S3_BUCKET_NAME}/${fileKey}`,
        tableData: tableData
      },
    }).promise();

    alert('File successfully uploaded and recorded!');
  } catch (error) {
    console.error('Upload error:', error);
    alert('File upload failed.');
  } finally {
    isUploading.value = false;
  }
};
</script>

<style scoped>
.upload-container {
  margin-top: 20px;
}

.upload-button {
  display: inline-block;
  padding: 10px 20px;
  background-color: #42b983;
  color: white;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.upload-button:hover {
  background-color: #36976d;
}

.is-loading {
  background-color: #999;
  cursor: not-allowed;
}
</style>
