<template>
  <div class="filter-bar">
    <div v-for="(step, index) in steps" :key="index" class="step-container">
      <div
        class="step-box"
        :class="{
          'current-step': index + 1 === currentStep,
          'completed-step': index + 1 < currentStep,
          'pending-step': index + 1 > currentStep
        }"
      >
        <span class="step-number">{{ index + 1 }}</span>
        <span class="step-name">{{ step }}</span>
      </div>
      <div
        v-if="index < steps.length - 1"
        class="step-connector"
        :class="{
          'completed-connector': index + 1 < currentStep,
          'pending-connector': index + 1 >= currentStep
        }"
      ></div>
    </div>
  </div>
</template>

<script setup>
defineProps({
  steps: {
    type: Array,
    required: true,
    default: () => []
  },
  currentStep: {
    type: Number,
    required: true,
    default: 1
  }
});
</script>

<style scoped>
.filter-bar {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  width: 100%;
  box-sizing: border-box;
}

.step-container {
  display: flex;
  align-items: center;
  width: 100%;
  max-width: 800px;
  margin: 0 auto;
}

.step-box {
  min-width: 40px;
  padding: 8px 12px;
  border: 2px solid #ccc;
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  position: relative;
  transition: all 0.3s ease;
}

.step-number {
  font-size: 16px;
  font-weight: bold;
  margin-bottom: 4px;
}

.step-name {
  font-size: 12px;
  white-space: nowrap;
  text-align: center;
}

.current-step {
  border-color: #ff9800;
  color: #ff9800;
  background-color: rgba(255, 152, 0, 0.1);
  transform: scale(1.05);
}

.completed-step {
  border-color: #4caf50;
  color: #4caf50;
  background-color: rgba(76, 175, 80, 0.1);
}

.pending-step {
  border-color: #e0e0e0;
  color: #9e9e9e;
  background-color: rgba(224, 224, 224, 0.1);
}

.step-connector {
  min-width: 40px;
  height: 2px;
  margin: 0 8px;
  transition: all 0.3s ease;
}

.completed-connector {
  background-color: #4caf50;
  box-shadow: 0 0 4px rgba(76, 175, 80, 0.3);
}

.pending-connector {
  background-color: #e0e0e0;
}

  .filter-bar {
    padding: 20px 5%;
    width: 90%;
    flex-direction: column;
    gap: 5px;
  }

  .step-container {
    flex-direction: column;
    gap: 15px;
  }

  .step-box {
    display: flex;
    flex-direction: row;
    min-width: auto;
    padding: 8px;
    width: 100%;
    justify-content: center;
    align-items: center;
  }

  .step-number {
    margin-bottom: 0;
    margin-right: 8px;
    font-size: 14px;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .step-name {
    font-size: 14px;
    text-align: center;
  }

  .step-connector {
    display: none;
  }

</style> 