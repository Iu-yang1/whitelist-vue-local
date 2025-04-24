<template>
  <div class="step4-container">
    <div class="progress-container">
      <FilterBar
        :steps="['加入须知', '白名单填写', '等待审核', '下载整合包']"
        :currentStep="4"
      />
    </div>

    <div class="content-container">
      <div class="title">获取整合包</div>

      <el-button
        type="primary"
        class="download-button"
        @click="startDownload"
        :disabled="isDownloading"
      >
        <el-icon><Download /></el-icon>
        {{ isDownloading ? '下载中...' : '立即下载' }}
      </el-button>

      <div v-if="isDownloading" class="download-message">
        <el-icon class="message-icon"><CircleCheck /></el-icon>
        <span>已开始下载</span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import FilterBar from './common/FilterBar.vue';
import { Download, CircleCheck } from '@element-plus/icons-vue';

const isDownloading = ref(false);

const startDownload = () => {
  isDownloading.value = true;
};
</script>

<style scoped>
.step4-container {
  display: flex;
  flex-direction: column;
  position: relative;
}

.progress-container {
  margin: 40px auto;
  width: 100%;
  max-width: 800px;
}

.content-container {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 20px;
  gap: 30px;
}

.title {
  font-size: 24px;
  font-weight: 500;
  color: var(--el-text-color-primary);
  margin-bottom: 20px;
}

.download-button {
  width: 200px;
  height: 50px;
  font-size: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  border-radius: 8px;
  transition: all 0.3s ease;
}

.download-button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(64, 158, 255, 0.2);
}

.download-message {
  display: flex;
  align-items: center;
  gap: 10px;
  color: #67C23A;
  font-size: 16px;
  font-weight: 500;
  padding: 15px 30px;
  background: rgba(103, 194, 58, 0.1);
  border-radius: 8px;
  animation: fadeIn 0.5s ease-out;
}

.message-icon {
  font-size: 20px;
}

/* 暗色模式适配 */
html.dark .title {
  color: var(--el-text-color-primary);
}

html.dark .download-message {
  background: rgba(103, 194, 58, 0.15);
}

/* 移动端适配 */
@media (max-width: 768px) {
  .progress-container {
    margin: 20px auto;
    padding: 0 20px;
  }

  .title {
    font-size: 20px;
  }

  .download-button {
    width: 90%;
    max-width: 300px;
  }

  .download-message {
    font-size: 14px;
    padding: 12px 24px;
  }
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style> 