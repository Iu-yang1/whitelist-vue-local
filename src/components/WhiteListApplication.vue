<template>
  <div
      class="app-wrapper"
      @click.self="handleFormBlur"
      @touchstart.self="handleFormBlur"
  >
    <SakuraBackground
        :currentTheme="currentTheme"
    />
    <div class="main-content">
      <div class="progress-container">
        <FilterBar
          :steps="['加入须知', '白名单填写', '等待审核', '下载整合包']"
          :currentStep="2"
        />
      </div>
      <div
          v-if="!isMobile"
          class="server-status-container"
          :class="{ 'form-focused': isFormFocused }"
      >
        <div class="status-header">
          <div class="header-title">
            <i class="el-icon-monitor"></i>
            <span>服务器状态</span>
            <el-button
                :loading="loading"
                class="refresh-btn"
                size="small"
                @click="debouncedRefresh(true)"
            >
              <el-icon>
                <Refresh/>
              </el-icon>
            </el-button>
          </div>
        </div>
        <div class="status-content">
          <div v-if="initialLoading" class="loading-state">
            <el-icon class="loading-icon">
              <Loading/>
            </el-icon>
            <span>加载中...</span>
          </div>

          <template v-else>
            <div v-for="(server, index) in displayedServers"
                 :key="server.name"
                 class="server-block animate-in">
              <div class="server-name">
                <i class="el-icon-connection"></i>
                {{ server.name }}
              </div>
              <div class="online-info">
                <div v-if="server.players.length > 0" class="player-list">
                  <div class="players-label">
                    <i class="el-icon-user"></i>
                    在线玩家 ({{ server.playerCount }})：
                  </div>
                  <div class="players-container">
                    <el-tag
                        v-for="player in server.players"
                        :key="player"
                        class="player-tag"
                        effect="light"
                        size="small"
                    >
                      {{ player }}
                    </el-tag>
                  </div>
                </div>
              </div>
            </div>

            <div v-if="showViewMore" class="view-more-container">
              <el-button
                  class="view-more-btn"
                  type="primary"
                  text
                  @click="$router.push('/server-status')"
              >
                <el-icon><ArrowRight /></el-icon>
                查看更多服务器
              </el-button>
            </div>

            <div class="query-time animate-in">
              <i class="el-icon-time"></i>
              {{ serverStatus.queryTime }}
            </div>
          </template>
        </div>
      </div>

      <div
          class="form-container"
          :class="{ 'focused': isFormFocused }"
          @click="handleFormFocus"
          @touchstart="handleFormFocus"
      >
        <div class="title-container">
          <i class="el-icon-user-solid"></i>
          <h2>白名单申请</h2>
          <el-button
              class="view-members-btn"
              text
              type="primary"
              @click="$router.push('/whitelist-members')"
          >
            <el-icon>
              <User/>
            </el-icon>
            查看成员
          </el-button>
        </div>

        <div class="description">
          欢迎加入我们的服务器！请填写以下信息完成白名单申请。
        </div>

        <el-form :model="form" class="animated-form" label-width="auto">
          <el-form-item label="ID：">
            <el-input v-model="form.userName" placeholder="请输入游戏名称"></el-input>
          </el-form-item>
          <el-form-item label="QQ：">
            <el-input v-model="form.qqNum" placeholder="请输入QQ号"></el-input>
          </el-form-item>
          <el-form-item label="正版：">
            <el-radio-group v-model="form.onlineFlag">
              <el-radio label="1">是</el-radio>
              <el-radio label="0">否</el-radio>
            </el-radio-group>
          </el-form-item>
          <el-form-item label="描述：" label-width="auto">
            <el-input v-model="form.remark" placeholder="请输入描述 非必填" type="textarea"></el-input>
          </el-form-item>
          <div class="checkbox-group">
            <el-checkbox v-model="form.checkedItems.joinedGroup">
              我已加群
            </el-checkbox>
            <el-checkbox v-model="form.checkedItems.readManual">
              <span class="clickable-text" @click="readManual">我已阅读手册</span>
            </el-checkbox>
            <el-checkbox v-model="form.checkedItems.knowIP">
              <span class="clickable-text" @click="showIPInfo">已知晓ip</span>
            </el-checkbox>
          </div>
          <div class="button-group">
            <el-button v-loading.fullscreen.lock="fullscreenLoading"
                       class="submit-btn"
                       type="primary"
                       @click="submitForm">
              <i class="el-icon-check"></i> 提交申请
            </el-button>
          </div>
        </el-form>
      </div>
    </div>
  </div>
</template>

<script setup>
import {computed, onMounted, onUnmounted, reactive, ref} from 'vue';
import {ElMessage} from 'element-plus';
import axios from 'axios';
import {ArrowRight, Loading, Refresh, User} from '@element-plus/icons-vue'
import {debounce} from 'lodash-es';
import SakuraBackground from './common/SakuraBackground.vue'
import FilterBar from './common/FilterBar.vue';
import { useRouter } from 'vue-router';
import {WIKI} from "@/config/links.js";

const http = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'https://application.shenzhuo.vip', // 使用环境变量
  timeout: 8000
});

const form = reactive({
  userName: '',
  qqNum: '',
  onlineFlag: '',
  remark: '',
  checkedItems: {
    joinedGroup: false,
    readManual: false,
    knowIP: false
  }
});

let loading = false;

const serverStatus = reactive({
  servers: [], // 存储所有服务器的状态
  queryTime: '-'
});

// 添加全屏loading状态
const fullscreenLoading = ref(false);

// 添加防抖处理的刷新函数
const debouncedRefresh = debounce((refresh) => {
  getOnlinePlayer(refresh);
}, 500);

// 添加初始加载状态
const initialLoading = ref(true);

const router = useRouter();

const submitForm = () => {
  // 验证必填字段
  if (!form.userName || !form.qqNum || !form.onlineFlag) {
    ElMessage.error('请填写完整信息');
    return;
  }

  // 验证QQ号格式
  if (!/^\d{5,11}$/.test(form.qqNum)) {
    ElMessage.error('QQ号格式错误');
    return;
  }

  // 验证复选框
  if (!form.checkedItems.joinedGroup) {
    ElMessage.error('请确认已加入群组');
    return;
  }
  if (!form.checkedItems.readManual) {
    ElMessage.error('请确认已阅读手册');
    return;
  }
  if (!form.checkedItems.knowIP) {
    ElMessage.error('请确认已知晓服务器IP');
    return;
  }

  fullscreenLoading.value = true;

  const headers = {};
  // 尝试获取用户IP，添加备用接口
  const getIpFromPrimarySource = () => {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), 3000);

      return fetch('https://ip.useragentinfo.com/json', {
        signal: controller.signal
      })
        .then(response => response.json())
        .catch(error => {
          console.warn('主要IP获取接口失败，尝试备用接口:', error);
          return getIpFromBackupSource();
        });
  };

  // 备用IP获取接口
  const getIpFromBackupSource = () => {
    return fetch('https://ipinfo.io/json')
        .then(response => response.json())
        .catch(error => {
          console.warn('备用IP获取接口也失败:', error);
          // 返回一个空对象，表示无法获取IP
          return {};
        });
  };

  // 开始获取IP
  getIpFromPrimarySource()
      .then(data => {
        // 如果成功获取到IP，添加到请求头
        if (data && data.ip) {
          headers['X-Real-IP'] = data.ip;
        }

        // 发送表单请求
        return http.post('/mc/whitelist/apply', form, {headers});
      })
      .then((res) => {
        if (res.data.code === 200) {
          ElMessage.success(res.data.msg);
          // 提交成功后跳转到第三步
          router.push('/whitelist-step3');
        } else {
          ElMessage.error(res.data.msg || '未知错误，请联系管理员');
        }
        fullscreenLoading.value = false;
      })
      .catch((error) => {
        console.error('提交表单请求出错：', error);
        ElMessage.error('提交表单时发生错误，请检查网络或联系管理员');
        fullscreenLoading.value = false;
      });
};

const getOnlinePlayer = (reflash) => {
  loading = true;
  http.get('/api/v1/getOnlinePlayer').then((res) => {
    if (res.data.code === 200) {
      const data = res.data.data;
      // 重置服务器列表
      serverStatus.servers = [];

      // 遍历所有服务器数据
      Object.entries(data).forEach(([serverName, serverData]) => {
        // 跳过查询时间字段
        if (serverName === '查询时间') {
          serverStatus.queryTime = serverData;
          return;
        }

        // 处理服务器数据
        try {
          // 处理在线玩家字符串
          let players = [];
          const playersStr = serverData['在线玩家'];
          if (playersStr) {
            players = playersStr.replace(/^\[|\]$/g, '').split(',')
                .map(p => p.trim())
                .filter(p => p);
          }

          serverStatus.servers.push({
            name: serverName,
            playerCount: serverData['在线人数'],
            players: players
          });
        } catch (e) {
          console.error(`处理服务器 ${serverName} 数据失败:`, e);
        }
      });
      if (reflash) {
        ElMessage.success('刷新成功！');
      }
    } else {
      ElMessage.error(res.data.msg || '获取服务器状态失败');
    }
    loading = false;
    initialLoading.value = false; // 设置初始加载状态为 false
  }).catch((error) => {
    console.error('查询在线玩家请求出错：', error);
    ElMessage.error('查询在线玩家时发生错误，请检查网络或联系管理员');
    loading = false;
    initialLoading.value = false; // 错误时也需要关闭加载状态
  });
};

// 获取当前主题
const currentTheme = ref(localStorage.getItem('theme') || 'default')

const isFormFocused = ref(false);

const handleFormFocus = () => {
  isFormFocused.value = true;
};

const handleFormBlur = () => {
  isFormFocused.value = false;
};

// 计算是否显示查看更多按钮
const showViewMore = computed(() => {
  return serverStatus.servers.length > 3;
});

// 计算要显示的服务器列表
const displayedServers = computed(() => {
  return serverStatus.servers.slice(0, 3);
});

// 检测是否为移动设备
const isMobile = ref(window.innerWidth <= 768);

// 添加窗口大小变化监听
const handleResize = () => {
  isMobile.value = window.innerWidth <= 768;
};

onMounted(() => {
  getOnlinePlayer();
  window.addEventListener('resize', handleResize);
});

// 组件卸载时移除事件监听
onUnmounted(() => {
  window.removeEventListener('resize', handleResize);
});

const readManual = () => {
  window.open(WIKI);
};

const showIPInfo = () => {
  window.open(WIKI);
};
</script>

<style scoped>
.app-wrapper {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  width: 100%;
  min-height: 100vh;
  background-size: 400% 400%;
  background-image: var(--theme-gradient, linear-gradient(-45deg, #e0e0ff, #ffd6e7, #ffdfd0, #fff4d1, #d4ffe6, #cce9ff, #f0e6ff, #ffe6f0, #ffe6e6));
  animation: warmGradient 20s ease infinite;
  font-family: 'CustomFont', sans-serif;
  transition: background-image 0.5s ease;
  padding: 0 20px;
  padding-bottom: 40px;
  box-sizing: border-box;
}

.main-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  max-width: 500px;
  position: relative;
}

.progress-container {
  width: 100%;
  margin: 0 auto;
  padding: 0 5%;
  box-sizing: border-box;
  display: flex;
  justify-content: center;
  align-items: center;
}

.form-container {
  background: var(--theme-bg);
  color: var(--theme-text);
  backdrop-filter: blur(8px);
  padding: 30px;
  border-radius: 24px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  transform: translateY(0);
  transition: all 0.3s ease;
  position: relative;
  z-index: 1001;
  margin: 0 10px;
  animation: fadeIn 0.6s ease-out;
}

.form-container:hover {
  transform: translateY(-5px);
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.15);
}

.form-container.focused {
  transform: translateY(-10px);
  box-shadow: 0 20px 40px rgba(var(--theme-primary-rgb), 0.2);
  transition: transform 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.title-container {
  text-align: center;
  margin-bottom: 25px;
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 0 10px;
}

.title-container h2 {
  color: var(--theme-primary);
  font-size: 28px;
  margin: 10px 0;
  flex-grow: 0; /* 防止标题占据过多空间 */
}

.description {
  text-align: center;
  color: #333;
  margin-bottom: 25px;
  font-size: 14px;
  font-weight: 500;
}

.animated-form :deep(.el-form-item) {
  transition: all 0.3s ease;
  margin-bottom: 25px;
}

.animated-form :deep(.el-form-item:hover) {
  transform: translateX(5px);
}

.button-group {
  display: flex;
  justify-content: center;
  margin-top: 30px;
}

.submit-btn {
  min-width: 140px;
  height: 40px;
  border-radius: 20px;
  font-weight: 500;
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
  font-family: 'CustomFont', sans-serif;
  background: linear-gradient(45deg, #409EFF, #36cfc9);
  border: none;
  box-shadow: 0 4px 15px rgba(64, 158, 255, 0.3);
  z-index: 1;
}

.submit-btn::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(
      90deg,
      transparent,
      rgba(255, 255, 255, 0.2),
      transparent
  );
  transition: 0.5s;
  z-index: -1;
}

.submit-btn:hover::before {
  left: 100%;
}

.submit-btn:hover {
  transform: scale(1.05);
  box-shadow: 0 6px 20px rgba(64, 158, 255, 0.4);
}

.submit-btn:active {
  transform: scale(0.98);
}

:deep(.el-form-item__label) {
  font-weight: 500;
  color: #303133;
  transition: all 0.3s ease;
}

:deep(.el-form-item:hover .el-form-item__label) {
  color: var(--theme-primary);
  transform: translateX(2px);
}

:deep(.el-input__inner) {
  border-radius: 8px;
  border: 1px solid rgba(64, 158, 255, 0.2);
  transition: all 0.3s ease;
  background: rgba(255, 255, 255, 0.8);
  height: 40px;
  transform: translateY(0);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  font-family: 'CustomFont', sans-serif;
}

:deep(.el-input__inner:hover) {
  border-color: #409EFF;
  box-shadow: 0 2px 8px rgba(64, 158, 255, 0.1);
}

:deep(.el-input__inner:focus) {
  border-color: #409EFF;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.1);
  background: #fff;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(64, 158, 255, 0.15);
}

:deep(.el-textarea__inner) {
  border-radius: 8px;
  border: 1px solid rgba(64, 158, 255, 0.2);
  transition: all 0.3s ease;
  background: rgba(255, 255, 255, 0.8);
  min-height: 100px;
  padding: 12px;
}

:deep(.el-textarea__inner:hover) {
  border-color: #409EFF;
}

:deep(.el-textarea__inner:focus) {
  border-color: #409EFF;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.1);
  background: #fff;
}

.form-container :deep(.el-input__wrapper),
.form-container :deep(.el-textarea__wrapper) {
  box-shadow: none !important;
  padding: 0;
  background: none;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.server-status-container {
  position: fixed;
  width: 280px;
  background: var(--theme-bg);
  color: var(--theme-text);
  backdrop-filter: blur(12px);
  border: 1px solid var(--theme-border);
  border-radius: 12px;
  padding: 15px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.08);
  transition: all 0.3s ease;
  z-index: 1001;

  /* 默认在右上角 */
  top: 20px;
  right: 20px;
}

.server-status-container:hover {
  transform: translateX(-5px);
  box-shadow: 4px 8px 32px rgba(0, 0, 0, 0.12);
}

.status-header {
  display: flex;
  align-items: center;
  margin-bottom: 12px;
  padding-bottom: 8px;
  border-bottom: 1px solid var(--theme-border);
}

.header-title {
  display: flex;
  align-items: center;
  font-weight: 500;
  color: var(--theme-primary);
}

.header-title i {
  margin-right: 8px;
}

.status-content {
  font-size: 14px;
  padding: 0 8px;
  width: 100%;
  box-sizing: border-box;
}

.server-name {
  font-weight: 500;
  color: var(--theme-primary);
  margin-bottom: 12px;
  display: flex;
  align-items: center;
  font-size: 15px;
  padding: 4px 8px;
  background: rgba(var(--theme-primary-rgb), 0.1);
  border-radius: 6px;
}

.server-name i {
  margin-right: 8px;
  color: #409EFF;
  font-size: 16px;
}

.online-info {
  padding-left: 4px;
  width: 100%;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

.player-list {
  display: flex;
  flex-direction: column;
  width: 100%;
}

.players-label {
  color: #606266;
  font-size: 14px;
  display: flex;
  align-items: center;
  margin-bottom: 10px;
  padding: 4px 0;
  padding-left: 4px;
}

.players-label i {
  margin-right: 8px;
  color: #67C23A;
  font-size: 15px;
}

.players-container {
  display: inline-flex;
  flex-wrap: wrap;
  gap: 4px;
  padding-left: 8px;
  width: auto;
  min-width: 0;
}

.player-tag {
  background-color: rgba(var(--theme-secondary-rgb), 0.1);
  border: 1px solid var(--theme-border);
  color: var(--theme-text);
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  transition: all 0.3s ease;
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.player-tag:hover {
  background-color: rgba(var(--theme-primary-rgb), 0.15);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(var(--theme-primary-rgb), 0.15);
}

/* 在线指示点 */
.online-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background-color: var(--theme-secondary);
  display: inline-block;
  margin-left: 4px;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0% {
    transform: scale(0.95);
    box-shadow: 0 0 0 0 rgba(var(--theme-secondary-rgb), 0.4);
  }
  70% {
    transform: scale(1);
    box-shadow: 0 0 0 6px rgba(var(--theme-secondary-rgb), 0);
  }
  100% {
    transform: scale(0.95);
    box-shadow: 0 0 0 0 rgba(var(--theme-secondary-rgb), 0);
  }
}

.query-time {
  font-size: 12px;
  color: var(--theme-text);
  padding: 8px;
  background: rgba(255, 255, 255, 0.5);
  border-radius: 8px;
  margin-top: 15px;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0.6;
}

.query-time i {
  margin-right: 6px;
  color: #909399;
  font-size: 14px;
}

.refresh-btn {
  padding: 2px;
  height: 20px;
  width: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
  margin-left: 8px;
  background-color: #f4f4f5;
  border: none;
  transform: scale(1);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.refresh-btn:hover {
  transform: rotate(180deg);
  background-color: #ecf5ff;
  color: #409EFF;
}

.refresh-btn:focus {
  color: #409EFF;
  background-color: #ecf5ff;
}

.refresh-btn:active {
  transform: scale(0.95);
}

.server-block {
  transition: all 0.3s ease;
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 15px;
  background: rgba(255, 255, 255, 0.5);
  border: 1px solid rgba(255, 255, 255, 0.2);
  width: 100%;
  box-sizing: border-box;
  overflow: hidden;
}

.server-block:last-child {
  border-bottom: none;
  margin-bottom: 0;
}

.player-count i,
.players-label i {
  margin-right: 8px;
  flex-shrink: 0;
}

.header-title :deep(.refresh-btn) {
  padding: 2px;
  height: 24px;
  width: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
  margin-left: 8px;
  background: none;
  border: none;
  color: #909399;
}

.header-title :deep(.refresh-btn .el-icon) {
  font-size: 16px;
}

.header-title :deep(.refresh-btn:hover) {
  transform: rotate(180deg);
  background: none;
  color: #409EFF;
}

.header-title :deep(.refresh-btn:focus) {
  color: #409EFF;
  background: none;
  outline: none;
}

.header-title :deep(.refresh-btn i) {
  margin: 0;
}

/* 优化服务器块样式 */
.server-block {
  transition: all 0.3s ease;
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 15px;
  background: rgba(255, 255, 255, 0.5);
  border: 1px solid rgba(255, 255, 255, 0.2);
  width: 100%;
}

.server-block:hover {
  background: rgba(255, 255, 255, 0.8);
}

/* 优化玩家标签样式 */
.player-tag {
  transition: all 0.3s ease;
  border-radius: 10px;
  padding: 2px 10px;
  background-color: rgba(103, 194, 58, 0.1);
  border: 1px solid rgba(103, 194, 58, 0.2);
  color: #67C23A;
  font-size: 12px;
  line-height: 1.4;
  white-space: nowrap;
}

.player-tag:hover {
  background-color: rgba(103, 194, 58, 0.2);
  transform: translateY(-1px);
  box-shadow: 0 2px 4px rgba(103, 194, 58, 0.1);
}

/* 优化查询时间样式 */
.query-time {
  font-size: 12px;
  color: #909399;
  padding: 8px;
  background: rgba(255, 255, 255, 0.5);
  border-radius: 8px;
  margin-top: 15px;
}

/* 添加错误信息样式 */
.error-message {
  color: #f56c6c;
  font-size: 13px;
  display: flex;
  align-items: center;
  padding: 12px;
  background: rgba(245, 108, 108, 0.1);
  border-radius: 8px;
  margin: 8px 0;
  animation: shake 0.5s cubic-bezier(0.36, 0.07, 0.19, 0.97) both;
}

.error-message i {
  margin-right: 8px;
  font-size: 16px;
}

@keyframes shake {
  10%, 90% {
    transform: translateX(-1px);
  }
  20%, 80% {
    transform: translateX(2px);
  }
  30%, 50%, 70% {
    transform: translateX(-4px);
  }
  40%, 60% {
    transform: translateX(4px);
  }
}

/* 主题特定样式 */
[data-theme="sakura"] .player-tag {
  background-color: rgba(255, 105, 180, 0.1);
  border-color: rgba(255, 105, 180, 0.2);
  color: #d4317c;
}

[data-theme="sakura"] .player-tag:hover {
  background-color: rgba(255, 105, 180, 0.2);
  box-shadow: 0 4px 12px rgba(255, 105, 180, 0.2);
  text-shadow: 0 0 8px rgba(255, 105, 180, 0.3);
}

[data-theme="cyberpunk"] .player-tag {
  background: rgba(0, 255, 221, 0.1);
  border-color: rgba(246, 24, 246, 0.3);
  color: #00ffd5;
  box-shadow: var(--theme-neon-shadow);
  text-shadow: var(--theme-text-shadow);
}

[data-theme="cyberpunk"] .player-tag:hover {
  background: rgba(0, 255, 221, 0.2);
  box-shadow: 0 0 15px rgba(0, 255, 221, 0.4);
}

/* 暗色模式样式 */
.dark .player-tag {
  background-color: rgba(255, 255, 255, 0.1);
  border-color: var(--theme-border-dark);
  color: var(--theme-text-dark);
}

.dark .player-tag:hover {
  background-color: rgba(255, 255, 255, 0.15);
  border-color: var(--theme-border-dark);
}

/* 在线玩家标签特殊样式 */
.player-tag.el-tag--success {
  background-color: rgba(var(--theme-secondary-rgb), 0.15);
  border-color: var(--theme-secondary);
  color: var(--theme-secondary);
}

.player-tag.el-tag--success:hover {
  background-color: rgba(var(--theme-secondary-rgb), 0.25);
  box-shadow: 0 4px 12px rgba(var(--theme-secondary-rgb), 0.2);
}

/* 修改森林主题下的标签样式，提高可读性 */
[data-theme="forest"] .player-tag {
  background-color: rgba(255, 255, 255, 0.15);
  border-color: rgba(144, 238, 144, 0.3);
  color: #2c5530; /* 深绿色文字 */
  font-weight: 500; /* 加粗文字 */
}

[data-theme="forest"] .player-tag:hover {
  background-color: rgba(255, 255, 255, 0.25);
  border-color: rgba(144, 238, 144, 0.4);
  box-shadow: 0 4px 12px rgba(144, 238, 144, 0.2);
}

/* 暗色模式下的森林主题 */
.dark[data-theme="forest"] .player-tag {
  background-color: rgba(144, 238, 144, 0.15);
  border-color: rgba(144, 238, 144, 0.3);
  color: #90EE90; /* 亮绿色文字 */
}

.dark[data-theme="forest"] .player-tag:hover {
  background-color: rgba(144, 238, 144, 0.25);
  box-shadow: 0 4px 12px rgba(144, 238, 144, 0.15);
}

/* 添加海洋主题特定样式 */
[data-theme="ocean"] .form-container {
  background: rgba(240, 248, 255, 0.95);
  border-color: rgba(100, 181, 246, 0.3);
  backdrop-filter: blur(20px);
  box-shadow: 0 8px 32px rgba(25, 118, 210, 0.15),
  0 2px 8px rgba(25, 118, 210, 0.1);
}

[data-theme="ocean"] .title-container h2 {
  color: #1976D2;
  text-shadow: 0 0 10px rgba(25, 118, 210, 0.2);
}

[data-theme="ocean"] .description {
  color: #0D47A1;
}

[data-theme="ocean"] .server-status-container {
  background: rgba(240, 248, 255, 0.95);
  border-color: rgba(100, 181, 246, 0.3);
  backdrop-filter: blur(20px);
  box-shadow: 0 8px 32px rgba(25, 118, 210, 0.15),
  0 2px 8px rgba(25, 118, 210, 0.1);
}

[data-theme="ocean"] .server-name {
  color: #1976D2;
  background: rgba(25, 118, 210, 0.1);
}

[data-theme="ocean"] .player-tag {
  background: rgba(3, 169, 244, 0.1);
  border-color: rgba(3, 169, 244, 0.3);
  color: #0277BD;
  font-weight: 500;
}

[data-theme="ocean"] .player-tag:hover {
  background: rgba(3, 169, 244, 0.2);
  box-shadow: 0 4px 12px rgba(3, 169, 244, 0.2);
  text-shadow: 0 0 8px rgba(3, 169, 244, 0.3);
}

[data-theme="ocean"] .header-title {
  color: #1976D2;
}

[data-theme="ocean"] .view-members-btn {
  color: #1976D2;
}

[data-theme="ocean"] .view-members-btn:hover {
  color: #0277BD;
  text-shadow: 0 0 8px rgba(3, 169, 244, 0.3);
}

/* 暗色模式下的海洋主题 */
.dark[data-theme="ocean"] .form-container {
  background: rgba(13, 71, 161, 0.85);
  border-color: rgba(100, 181, 246, 0.2);
  backdrop-filter: blur(20px);
  box-shadow: 0 8px 32px rgba(13, 71, 161, 0.3),
  0 2px 8px rgba(13, 71, 161, 0.2);
}

.dark[data-theme="ocean"] .server-status-container {
  background: rgba(13, 71, 161, 0.85);
  border-color: rgba(100, 181, 246, 0.2);
  backdrop-filter: blur(20px);
  box-shadow: 0 8px 32px rgba(13, 71, 161, 0.3),
  0 2px 8px rgba(13, 71, 161, 0.2);
}

.dark[data-theme="ocean"] .title-container h2 {
  color: #64B5F6;
  text-shadow: 0 0 10px rgba(100, 181, 246, 0.3);
}

.dark[data-theme="ocean"] .description {
  color: #90CAF9;
}

.dark[data-theme="ocean"] .server-name {
  color: #64B5F6;
  background: rgba(100, 181, 246, 0.15);
}

.dark[data-theme="ocean"] .player-tag {
  background: rgba(3, 169, 244, 0.15);
  border-color: rgba(3, 169, 244, 0.3);
  color: #81D4FA;
}

.dark[data-theme="ocean"] .player-tag:hover {
  background: rgba(3, 169, 244, 0.25);
  box-shadow: 0 4px 12px rgba(3, 169, 244, 0.2);
}

.dark[data-theme="ocean"] .header-title {
  color: #64B5F6;
}

.dark[data-theme="ocean"] .view-members-btn {
  color: #64B5F6;
}

.dark[data-theme="ocean"] .view-members-btn:hover {
  color: #81D4FA;
}

/* 添加极光主题特定样式 */
[data-theme="aurora"] .form-container {
  background: rgba(224, 247, 250, 0.95);
  border-color: rgba(77, 208, 225, 0.3);
  backdrop-filter: blur(20px);
  box-shadow: 0 8px 32px rgba(38, 198, 218, 0.15),
  0 2px 8px rgba(38, 198, 218, 0.1);
}

[data-theme="aurora"] .title-container h2 {
  color: #00838F;
  text-shadow: 0 0 15px rgba(38, 198, 218, 0.3);
}

[data-theme="aurora"] .description {
  color: #006064;
}

[data-theme="aurora"] .server-status-container {
  background: rgba(224, 247, 250, 0.95);
  border-color: rgba(77, 208, 225, 0.3);
  backdrop-filter: blur(20px);
  box-shadow: 0 8px 32px rgba(38, 198, 218, 0.15),
  0 2px 8px rgba(38, 198, 218, 0.1);
}

[data-theme="aurora"] .server-name {
  color: #00838F;
  background: rgba(38, 198, 218, 0.1);
}

[data-theme="aurora"] .player-tag {
  background: rgba(38, 198, 218, 0.1);
  border-color: rgba(0, 191, 165, 0.3);
  color: #00838F;
  font-weight: 500;
}

[data-theme="aurora"] .player-tag:hover {
  background: rgba(38, 198, 218, 0.2);
  box-shadow: 0 4px 12px rgba(38, 198, 218, 0.2);
  text-shadow: 0 0 8px rgba(38, 198, 218, 0.3);
}

[data-theme="aurora"] .header-title {
  color: #00838F;
}

[data-theme="aurora"] .view-members-btn {
  color: #00838F;
}

[data-theme="aurora"] .view-members-btn:hover {
  color: #00ACC1;
  text-shadow: 0 0 8px rgba(38, 198, 218, 0.3);
}

/* 暗色模式下的极光主题 */
.dark[data-theme="aurora"] .form-container {
  background: rgba(0, 96, 100, 0.85);
  border-color: rgba(77, 208, 225, 0.2);
  backdrop-filter: blur(20px);
  box-shadow: 0 8px 32px rgba(0, 96, 100, 0.3),
  0 2px 8px rgba(0, 96, 100, 0.2);
}

.dark[data-theme="aurora"] .server-status-container {
  background: rgba(0, 96, 100, 0.85);
  border-color: rgba(77, 208, 225, 0.2);
  backdrop-filter: blur(20px);
  box-shadow: 0 8px 32px rgba(0, 96, 100, 0.3),
  0 2px 8px rgba(0, 96, 100, 0.2);
}

.dark[data-theme="aurora"] .title-container h2 {
  color: #4DD0E1;
  text-shadow: 0 0 15px rgba(77, 208, 225, 0.4);
}

.dark[data-theme="aurora"] .description {
  color: #B2EBF2;
}

.dark[data-theme="aurora"] .server-name {
  color: #4DD0E1;
  background: rgba(77, 208, 225, 0.15);
}

.dark[data-theme="aurora"] .player-tag {
  background: rgba(38, 198, 218, 0.15);
  border-color: rgba(0, 191, 165, 0.3);
  color: #80DEEA;
}

.dark[data-theme="aurora"] .player-tag:hover {
  background: rgba(38, 198, 218, 0.25);
  box-shadow: 0 4px 12px rgba(38, 198, 218, 0.2);
}

.dark[data-theme="aurora"] .header-title {
  color: #4DD0E1;
}

.dark[data-theme="aurora"] .view-members-btn {
  color: #4DD0E1;
}

.dark[data-theme="aurora"] .view-members-btn:hover {
  color: #80DEEA;
}

/* 添加加载状态样式 */
.loading-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 20px;
  color: var(--theme-text);
  gap: 10px;
}

.loading-icon {
  font-size: 24px;
  animation: rotate 1s linear infinite;
}

@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

/* 添加内容淡入动画 */
.animate-in {
  animation: fadeIn 0.3s ease-out;
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

/* 适配暗色模式 */
.dark .loading-state {
  color: var(--theme-text-dark);
}

/* 主题特定样式 */
[data-theme="sakura"] .loading-state {
  color: var(--theme-primary);
}

[data-theme="cyberpunk"] .loading-state {
  color: #00ffd5;
  text-shadow: var(--theme-text-shadow);
}

[data-theme="ocean"] .loading-state {
  color: #1976D2;
}

.dark[data-theme="ocean"] .loading-state {
  color: #64B5F6;
}

[data-theme="aurora"] .loading-state {
  color: #00838F;
}

.dark[data-theme="aurora"] .loading-state {
  color: #4DD0E1;
}

/* 添加查看更多按钮样式 */
.view-more-container {
  display: flex;
  justify-content: center;
  margin-top: 15px;
  padding-top: 15px;
  border-top: 1px solid var(--theme-border);
}

.view-more-btn {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 14px;
  color: var(--theme-primary);
  transition: all 0.3s ease;
  font-family: 'CustomFont', sans-serif;
}

.view-more-btn:hover {
  transform: translateX(4px);
}

/* 添加漂亮的加载和完成动画 */
@keyframes successPulse {
  0% {
    box-shadow: 0 0 0 0 rgba(103, 194, 58, 0.4);
    background-color: #67C23A;
  }
  70% {
    box-shadow: 0 0 0 15px rgba(103, 194, 58, 0);
    background-color: #67C23A;
  }
  100% {
    box-shadow: 0 0 0 0 rgba(103, 194, 58, 0);
    background-color: #67C23A;
  }
}

/* 添加输入框聚焦效果 */
.form-container .animated-form:focus-within {
  transform: scale(1.01);
}

/* 添加标题和描述的进场动画 */
.title-container, .description {
  animation: slideInFromTop 0.6s ease-out;
}

@keyframes slideInFromTop {
  0% {
    opacity: 0;
    transform: translateY(-20px);
  }
  100% {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 优化表单项进场动画 */
.animated-form :deep(.el-form-item) {
  animation: fadeInStaggered 0.5s ease-out backwards;
}

.animated-form :deep(.el-form-item:nth-child(1)) {
  animation-delay: 0.1s;
}

.animated-form :deep(.el-form-item:nth-child(2)) {
  animation-delay: 0.2s;
}

.animated-form :deep(.el-form-item:nth-child(3)) {
  animation-delay: 0.3s;
}

.animated-form :deep(.el-form-item:nth-child(4)) {
  animation-delay: 0.4s;
}

.button-group {
  animation: fadeInStaggered 0.5s ease-out backwards;
  animation-delay: 0.5s;
}

@keyframes fadeInStaggered {
  0% {
    opacity: 0;
    transform: translateX(-20px);
  }
  100% {
    opacity: 1;
    transform: translateX(0);
  }
}

/* 暗色模式输入框样式 */
html.dark :deep(.el-input__inner),
html.dark :deep(.el-textarea__inner) {
  background-color: rgba(30, 30, 40, 0.8) !important;
  border-color: rgba(255, 255, 255, 0.1) !important;
  color: rgba(255, 255, 255, 0.9) !important;
  box-shadow: none !important;
}

/* 输入框和文本框的悬停和焦点状态 */
html.dark :deep(.el-input__inner:hover),
html.dark :deep(.el-textarea__inner:hover) {
  border-color: rgba(64, 158, 255, 0.3) !important;
  background-color: rgba(35, 35, 45, 0.8) !important;
}

html.dark :deep(.el-input__inner:focus),
html.dark :deep(.el-textarea__inner:focus) {
  border-color: rgba(64, 158, 255, 0.5) !important;
  background-color: rgba(35, 35, 45, 0.9) !important;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2) !important;
}

/* 输入框和文本框的包装器样式 */
html.dark :deep(.el-input__wrapper),
html.dark :deep(.el-textarea__wrapper) {
  background-color: transparent !important;
  box-shadow: none !important;
}

html.dark :deep(.el-radio__label) {
  color: rgba(255, 255, 255, 0.9);
}

html.dark :deep(.el-form-item__label) {
  color: rgba(255, 255, 255, 0.9);
}

/* 更顺滑的表单聚焦过渡 */
.form-container.focused {
  transform: translateY(-10px);
  transition: transform 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.progress-container {
  margin: 40px auto;
  width: 100%;
  max-width: 800px;
}

.checkbox-group {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin: 20px 0;
  padding: 0 20px;
}

.clickable-text {
  color: #409EFF;
  cursor: pointer;
  text-decoration: underline;
  transition: all 0.3s ease;
}

.clickable-text:hover {
  color: #66b1ff;
}

/* 暗色模式适配 */
html.dark .clickable-text {
  color: #64B5F6;
}

html.dark .clickable-text:hover {
  color: #90CAF9;
}

/* 移动端适配 */
@media (max-width: 768px) {
  .progress-container {
    margin: 20px auto;
    padding: 0 20px;
  }

  .checkbox-group {
    padding: 0 10px;
  }
}
</style>
