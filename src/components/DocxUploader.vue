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

const handleFileUpload = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  if (file.type !== 'application/vnd.openxmlformats-officedocument.wordprocessingml.document') {
    alert('Please upload a valid .docx file');
    return;
  }

  isUploading.value = true;

  try {

    // Prepare file upload to S3
    const fileKey = file.name;

    // Upload to S3
    await s3.upload({
      Bucket: import.meta.env.VITE_S3_BUCKET_NAME,
      Key: fileKey,
      Body: file,
      ContentType: file.type,
      ACL: 'private',
    }).promise();

    console.log('File uploaded to S3:', fileKey);

    // Send data to Lambda (via API Gateway) instead of DynamoDB
    const lambdaFunctionURL = import.meta.env.VITE_LAMBDA_URL

    const payload = {
      s3link: `https://s3.${import.meta.env.VITE_AWS_REGION}.amazonaws.com/${import.meta.env.VITE_S3_BUCKET_NAME}/${fileKey}`,
    };

    const response = await fetch(lambdaFunctionURL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(payload),
    });

    if (!response.ok) {
      throw new Error(`Lambda call failed: ${response.statusText}`);
    }

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
/* Upload button container positioned at the top but not fixed */
.upload-container {
  display: flex;
  justify-content: center; /* Center horizontally */
  align-items: center;
  width: 100%;
  margin-top: 20px; /* Adds space from the top of the page */
  margin-bottom: 20px; /* Adds spacing below */
}

/* Upload button styling */
.upload-button {
  padding: 12px 24px;
  background-color: #42b983;
  color: white;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
  font-size: 16px;
}

.upload-button:hover {
  background-color: #36976d;
}

/* Disabled upload button when loading */
.is-loading {
  background-color: #999;
  cursor: not-allowed;
}
</style>
