<template>
  <div class="server-status-page">
    <div class="header">
      <el-button class="back-btn" @click="$router.push('/')" text>
        <el-icon><Back /></el-icon>
        返回
      </el-button>
      <h2>服务器状态</h2>
      <el-button
        :loading="loading"
        class="refresh-btn"
        @click="refreshStatus"
      >
        <el-icon><Refresh /></el-icon>
        刷新
      </el-button>
    </div>

    <div class="servers-container">
      <div v-if="loading" class="loading-state">
        <el-icon class="loading-icon"><Loading /></el-icon>
        <span>加载中...</span>
      </div>

      <template v-else>
        <div v-for="server in servers"
             :key="server.name"
             class="server-card animate-in">
          <div class="server-header">
            <div class="server-name">
              <el-icon><Monitor /></el-icon>
              {{ server.name }}
            </div>
            <div class="player-count">
              <el-icon><User /></el-icon>
              在线: {{ server.playerCount }}
            </div>
          </div>

          <div v-if="server.players.length > 0" class="players-section">
            <div class="players-container">
              <el-tag
                v-for="player in server.players"
                :key="player"
                class="player-tag"
                effect="light"
              >
                {{ player }}
              </el-tag>
            </div>
          </div>
          <div v-else class="no-players">
            暂无在线玩家
          </div>
        </div>

        <div class="query-time">
          <el-icon><Timer /></el-icon>
          {{ queryTime }}
        </div>
      </template>
    </div>
  </div>
</template>

<script setup>
import {onMounted, ref} from 'vue'
import {ElMessage} from 'element-plus'
import axios from 'axios'
import {Back, Loading, Monitor, Refresh, Timer, User} from '@element-plus/icons-vue'

const loading = ref(false)
const servers = ref([])
const queryTime = ref('-')

const http = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'https://application.shenzhuo.vip',
  timeout: 8000
})

const refreshStatus = async () => {
  loading.value = true
  try {
    const res = await http.get('/api/v1/getOnlinePlayer')
    if (res.data.code === 200) {
      const data = res.data.data
      servers.value = []

      Object.entries(data).forEach(([serverName, serverData]) => {
        if (serverName === '查询时间') {
          queryTime.value = serverData
          return
        }

        let players = []
        const playersStr = serverData['在线玩家']
        if (playersStr) {
          players = playersStr.replace(/^\[|\]$/g, '').split(',')
            .map(p => p.trim())
            .filter(p => p)
        }

        servers.value.push({
          name: serverName,
          playerCount: serverData['在线人数'],
          players: players
        })
      })
      ElMessage.success('刷新成功')
    }
  } catch (error) {
    console.error('获取服务器状态失败：', error)
    ElMessage.error('获取服务器状态失败')
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  refreshStatus()
})
</script>

<style scoped>
.server-status-page {
  min-height: 100vh;
  padding: 20px;
  background-image: var(--theme-gradient);
  background-size: 400% 400%;
  animation: warmGradient 20s ease infinite;
  font-family: 'CustomFont', sans-serif;
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 30px;
  background: var(--theme-bg);
  padding: 15px 20px;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.header h2 {
  margin: 0;
  color: var(--theme-primary);
}

.servers-container {
  display: grid;
  gap: 20px;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
}

.server-card {
  background: var(--theme-bg);
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

.server-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
}

.server-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.server-name {
  font-size: 18px;
  font-weight: 500;
  color: var(--theme-primary);
  display: flex;
  align-items: center;
  gap: 8px;
}

.player-count {
  display: flex;
  align-items: center;
  gap: 6px;
  color: var(--theme-text);
}

.players-section {
  margin-top: 15px;
}

.players-container {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.player-tag {
  background-color: rgba(var(--theme-secondary-rgb), 0.1);
  border: 1px solid var(--theme-border);
  transition: all 0.3s ease;
}

.player-tag:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(var(--theme-secondary-rgb), 0.15);
}

.no-players {
  color: var(--theme-text);
  opacity: 0.6;
  text-align: center;
  padding: 10px;
}

.query-time {
  grid-column: 1 / -1;
  text-align: center;
  color: var(--theme-text);
  opacity: 0.6;
  margin-top: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}

.loading-state {
  grid-column: 1 / -1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 200px;
  gap: 10px;
}

.loading-icon {
  font-size: 24px;
  animation: rotate 1s linear infinite;
}

@keyframes rotate {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

@keyframes warmGradient {
  0% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}

/* 暗色模式适配 */
.dark .server-card {
  background: rgba(35, 35, 45, 0.7);
  backdrop-filter: blur(10px);
}

.dark .header {
  background: rgba(35, 35, 45, 0.7);
  backdrop-filter: blur(10px);
}

/* 响应式设计 */
@media (max-width: 768px) {
  .server-status-page {
    padding: 10px;
  }

  .servers-container {
    grid-template-columns: 1fr;
  }

  .header {
    flex-direction: row;
    padding: 10px 15px;
  }

  .server-card {
    margin-bottom: 15px;
  }
}
</style> 