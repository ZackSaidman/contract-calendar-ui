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

    // Log to DynamoDB
    await dynamoDB.put({
      TableName: import.meta.env.VITE_DYNAMO_TABLE_NAME,
      Item: {
        filename: file.name,
        date: '2025-03-02',
        s3link: fileKey
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
