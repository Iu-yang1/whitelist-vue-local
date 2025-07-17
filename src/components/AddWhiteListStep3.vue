<template>
  <div class="step3-container">
    <!-- 进度条 -->
    <div class="progress-container">
      <FilterBar
        :steps="['加入须知', '白名单填写', '等待审核', '下载整合包']"
        :currentStep="3"
      />
    </div>

    <!-- 主要内容区域 -->
    <div class="content-container">
      <!-- 提示文字 -->
      <div class="status-message">
        <el-icon class="status-icon"><CircleCheck /></el-icon>
        <span>审核已提交</span>
      </div>
      <div class="waiting-message">
        <p>您的申请已提交，请前往邮箱完成验证，验证通过后即可获得服务器访问权限</p>
      </div>

      <!-- 操作按钮组 -->
      <div class="action-buttons">
        <el-button type="primary" class="action-button" @click="queryWhitelist">
          <el-icon><Search /></el-icon>
          前往QQ邮箱验证
        </el-button>
<!--        <el-button type="info" class="action-button" @click="showFAQ">
          <el-icon><QuestionFilled /></el-icon>
          疑难解答
        </el-button>-->
      </div>
    </div>

    <!-- 底部操作栏 -->
    <div class="bottom-bar">
      <el-button type="primary" class="next-button" @click="goToNextStep">
        下一步
        <el-icon class="el-icon--right"><ArrowRight /></el-icon>
      </el-button>
    </div>
  </div>
</template>

<script setup>
import { useRouter } from 'vue-router';
import FilterBar from './common/FilterBar.vue';
import { CircleCheck, Search, QuestionFilled, ArrowRight } from '@element-plus/icons-vue';
import {QQ_MAIL} from "@/config/links.js";

const router = useRouter();

const queryWhitelist = () => {
  window.open(QQ_MAIL)
};

const showFAQ = () => {
};

const goToNextStep = () => {
  router.push('/whitelist-step4');
};
</script>

<style scoped>
.step3-container {
  display: flex;
  flex-direction: column;
  position: relative;
  padding-bottom: 80px;
  width: 100%;
  box-sizing: border-box;
}

.progress-container {
  margin: 40px auto;
  width: 100%;
  max-width: 800px;
  padding: 0 5%;
  box-sizing: border-box;
  display: flex;
  justify-content: center;
  align-items: center;
}

.content-container {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 20px;
  gap: 40px;
}

.status-message {
  display: flex;
  align-items: center;
  gap: 10px;
  color: #67C23A;
  font-size: 20px;
  font-weight: 500;
  padding: 20px 40px;
  background: rgba(103, 194, 58, 0.1);
  border-radius: 8px;
  animation: fadeIn 0.5s ease-out;
}

.status-icon {
  font-size: 24px;
}

.action-buttons {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
}

.action-button {
  width: 180px;
  height: 50px;
  font-size: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  border-radius: 8px;
  transition: all 0.3s ease;
  margin: 0;
}

.action-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.bottom-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  height: 80px;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  display: flex;
  justify-content: center;
  align-items: center;
  box-shadow: 0 -2px 10px rgba(0, 0, 0, 0.05);
  z-index: 1000;
}

.next-button {
  width: 200px;
  height: 50px;
  font-size: 16px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  transition: all 0.3s ease;
}

.next-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(64, 158, 255, 0.2);
}

/* 暗色模式适配 */
html.dark .bottom-bar {
  background: rgba(30, 30, 40, 0.95);
  box-shadow: 0 -2px 10px rgba(0, 0, 0, 0.2);
}

html.dark .status-message {
  background: rgba(103, 194, 58, 0.15);
}

/* 移动端适配 */
@media (max-width: 768px) {
  .progress-container {
    margin: 20px auto;
    padding: 0 5%;
  }

  .status-message {
    font-size: 18px;
    padding: 15px 30px;
  }

  .action-buttons {
    flex-direction: column;
    gap: 15px;
  }

  .action-button {
    width: 100%;
    max-width: 300px;
  }

  .next-button {
    width: 90%;
    max-width: 300px;
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