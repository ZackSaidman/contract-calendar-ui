<template>
    <div v-if="isVisible" class="modal-overlay" @click.self="close">
        <div class="modal-content">
            <slot></slot>
            <button class="close-btn" @click="close">Close</button>
        </div>
    </div>
    <!-- Debugging the modal visibility -->
    <div v-else class="debug-message">
        Modal is not visible
    </div>
</template>

<script setup>
const props = defineProps({
    isVisible: Boolean
});

const emit = defineEmits(['close']);

const close = () => {
    emit('close');
};
</script>

<style scoped>
.modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background-color: rgba(0, 0, 0, 0.7); /* Dark overlay */
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 9999; /* Ensure it stays on top */
}

.modal-content {
    background: white;
    padding: 20px;
    border-radius: 8px;
    max-width: 90%;
    max-height: 80%;
    overflow: auto;
    position: relative;
    display: flex;
    flex-direction: column; /* Ensure content is stacked vertically */
    padding-top: 40px; /* Extra space for the close button */
}

.close-btn {
    position: absolute;
    top: 10px;
    right: 10px;
    padding: 8px 16px;
    background-color: red;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    z-index: 10; /* Ensure it stays above the content */
}

.debug-message {
    color: red;
    text-align: center;
    font-size: 16px;
}
</style>
