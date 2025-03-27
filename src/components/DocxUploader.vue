<template>
<div class="upload-container">
    <label for="file-upload" class="upload-button">Upload DOCX</label>
    <input id="file-upload" type="file" accept=".docx" @change="handleFileUpload" hidden />
</div>
</template>

<script setup>
const handleFileUpload = (event) => {
const file = event.target.files[0];
if (file && file.type === 'application/vnd.openxmlformats-officedocument.wordprocessingml.document') {
    console.log('Uploaded file:', file);
    // Add your file handling logic here
} else {
    alert('Please upload a valid .docx file');
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
</style>
