<template>
  <div class="quiz-container">
    <div class="quiz-card">
      <h2 class="quiz-title">白名单验证题目</h2>
      <p class="quiz-subtitle">请完成以下问题以继续验证流程</p>

      <div v-if="loading" class="loading-state">
        <el-icon class="loading-icon">
          <Loading/>
        </el-icon>
        <span>正在加载题目...</span>
      </div>

      <div v-else-if="error" class="error-state">
        <el-icon class="error-icon">
          <Warning/>
        </el-icon>
        <span>{{ error }}</span>
        <el-button type="primary" @click="fetchQuestions">重试</el-button>
      </div>

      <div v-else class="questions-container">
        <form @submit.prevent="submitAnswers">
          <div v-for="question in questions" :key="question.id" class="question-item">
            <div class="question-text">
              {{ question.questionText }}
              <span v-if="question.isRequired === 1" class="required-marker">*</span>
            </div>

            <!-- 单选题 -->
            <div v-if="question.questionType === 1" class="question-options">
              <el-radio-group v-model="answers[question.id]">
                <el-radio
                    v-for="option in question.whitelistQuizAnswerVoList"
                    :key="option.id"
                    :label="option.id">
                  {{ option.answerText }}
                </el-radio>
              </el-radio-group>
            </div>

            <!-- 多选题 -->
            <div v-else-if="question.questionType === 2" class="question-options">
              <el-checkbox-group v-model="answers[question.id]">
                <el-checkbox
                    v-for="option in question.whitelistQuizAnswerVoList"
                    :key="option.id"
                    :label="option.id">
                  {{ option.answerText }}
                </el-checkbox>
              </el-checkbox-group>
            </div>

            <!-- 填空题 -->
            <div v-else-if="question.questionType === 3" class="question-options">
              <el-input
                  v-model="answers[question.id]"
                  :rows="3"
                  placeholder="请输入您的回答"
                  type="textarea">
              </el-input>
            </div>

            <!-- 数学验证题 -->
            <div v-else-if="question.questionType === 4" class="question-options">
              <el-input
                  v-model="answers[question.id]"
                  placeholder="请输入验证答案"
                  type="text">
              </el-input>
            </div>
          </div>

          <div class="actions">
            <el-button :loading="submitting" native-type="submit" type="primary">提交并继续验证</el-button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<script setup>
import {computed, onMounted, ref} from 'vue'
import {useRoute, useRouter} from 'vue-router'
import {Loading, Warning} from '@element-plus/icons-vue'
import axios from 'axios'
import {ElMessage} from 'element-plus'

const route = useRoute()
const router = useRouter()
const loading = ref(true)
const submitting = ref(false)
const error = ref(null)
const questions = ref([])
const answers = ref({})


// 验证码
const code = computed(() => route.query.code)

// 检查是否所有必答题目都已回答
const validateAnswers = () => {
  for (const question of questions.value) {
    if (question.isRequired === 1) {
      const answer = answers.value[question.id]
      if (!answer || (Array.isArray(answer) && answer.length === 0)) {
        ElMessage.error(`请回答必答题目: ${question.questionText}`)
        return false
      }
    }
  }
  return true
}

// 获取问题
const fetchQuestions = async () => {
  loading.value = true
  error.value = null

  if (!code.value) {
    error.value = '无效的验证链接'
    loading.value = false
    return
  }

  try {
    const apiUrl = import.meta.env.VITE_API_URL
    const res = await axios.get(`${apiUrl}/api/v1/getQuestions`, {
      params: {code: code.value},
      withCredentials: true
    })

    if (res.data.code === 200) {
      questions.value = res.data.data || []

      // 如果没有问题，直接跳转到验证页面
      if (!questions.value.length) {
        console.log('没有问卷题目，直接进入验证页面')
        router.push({
          path: '/verify',
          query: {code: code.value}
        })
        return
      }

      // 初始化答案对象
      questions.value.forEach(q => {
        if (q.questionType === 2) { // 多选题初始化为数组
          answers.value[q.id] = []
        } else {
          answers.value[q.id] = ''
        }
      })

      // 按sortOrder排序
      questions.value.sort((a, b) => a.sortOrder - b.sortOrder)
    } else {
      // 如果返回错误，检查是否是因为没有题目
      if (res.data.msg && res.data.msg.includes('没有问卷题目')) {
        console.log('API返回无问卷题目，直接进入验证页面')
        router.push({
          path: '/verify',
          query: {code: code.value}
        })
        return
      }

      error.value = res.data.msg || '获取题目失败'
    }
  } catch (err) {
    console.error('获取题目失败:', err)
    if (err.response && err.response.status === 404) {
      console.log('API返回404，可能没有问卷功能，直接进入验证页面')
      router.push({
        path: '/verify',
        query: {code: code.value}
      })
      return
    }

    error.value = '获取题目时出现错误，请重试'
  } finally {
    loading.value = false
  }
}

// 提交答案
const submitAnswers = async () => {
  if (!validateAnswers()) {
    return
  }

  submitting.value = true


  try {
    // 准备请求头
    const headers = {}

    // 获取用户IP
    try {
      const ipResponse = await fetch('https://api.ipify.org?format=json')
      const ipData = await ipResponse.json()
      if (ipData && ipData.ip) {
        headers['X-Real-IP'] = ipData.ip
      }
    } catch (ipErr) {
      console.warn('获取IP失败:', ipErr)
      // 备用IP获取方法
      try {
        const backupIpResponse = await fetch('https://ipinfo.io/json')
        const backupIpData = await backupIpResponse.json()
        if (backupIpData && backupIpData.ip) {
          headers['X-Real-IP'] = backupIpData.ip
        }
      } catch (backupIpErr) {
        console.warn('备用IP获取也失败:', backupIpErr)
      }
    }

    const apiUrl = import.meta.env.VITE_API_URL

    // 转换答案格式
    const formattedAnswers = Object.entries(answers.value).map(([questionId, answer]) => {
      const question = questions.value.find(q => q.id === parseInt(questionId))
      return {
        questionId: parseInt(questionId),
        answer: Array.isArray(answer) ? answer.join(',') : String(answer || ''),
        verificationId: question.questionType === 4 ? question.verificationId : undefined
      }
    })

    const res = await axios.post(`${apiUrl}/api/v1/submitQuiz`, {
      code: code.value,
      answers: formattedAnswers
    }, {
      headers,
      withCredentials: true
    })

    if (res.data.code === 200) {
      // 提交成功，进入验证页面
      router.push({
        path: '/verify',
        query: {code: code.value}
      })
    } else {
      ElMessage.error(res.data.msg || '提交答案失败')
    }
  } catch (err) {
    console.error('提交答案失败:', err)

    // 如果是网络错误或CORS错误，提供更具体的错误信息
    if (err.message && err.message.includes('Network Error')) {
      ElMessage.error('网络连接错误，请检查网络设置或联系管理员')
    } else if (err.message && err.message.includes('CORS')) {
      ElMessage.error('跨域请求错误，请联系管理员配置服务器访问权限')
    } else {
      ElMessage.error('提交过程中出现错误，请重试')
    }
  } finally {
    submitting.value = false
  }
}

onMounted(() => {
  fetchQuestions()
})
</script>

<style scoped>


.quiz-container {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background: var(--theme-gradient);
  padding: 20px;
  font-family: 'CustomFont', sans-serif;
}

.quiz-card {
  background: var(--theme-bg);
  padding: 40px;
  border-radius: 16px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  text-align: center;
  width: 100%;
  max-width: 800px;
  backdrop-filter: blur(8px);
  font-family: 'CustomFont', sans-serif;
}

.quiz-title {
  font-size: 24px;
  margin-bottom: 8px;
  color: var(--theme-primary);
  font-family: 'CustomFont', sans-serif;
}

.quiz-subtitle {
  color: var(--theme-text-secondary);
  margin-bottom: 32px;
  font-family: 'CustomFont', sans-serif;
}

.loading-state,
.error-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  margin: 32px 0;
  font-family: 'CustomFont', sans-serif;
}

.loading-icon {
  font-size: 48px;
  color: var(--theme-primary);
  animation: rotate 2s linear infinite;
}

.error-icon {
  font-size: 48px;
  color: #f56c6c;
}

.questions-container {
  text-align: left;
  font-family: 'CustomFont', sans-serif;
}

.question-item {
  margin-bottom: 32px;
}

.question-text {
  font-weight: 500;
  margin-bottom: 16px;
  color: var(--theme-text);
  font-family: 'CustomFont', sans-serif;
}

.required-marker {
  color: #f56c6c;
  margin-left: 4px;
}

.question-options {
  margin-left: 8px;
}

.actions {
  margin-top: 32px;
  display: flex;
  justify-content: center;
}

@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

/* 自定义 Element Plus 组件样式 */
:deep(.el-radio),
:deep(.el-checkbox),
:deep(.el-radio__label),
:deep(.el-checkbox__label) {
  font-family: 'CustomFont', sans-serif;
}

:deep(.el-input__inner),
:deep(.el-textarea__inner) {
  font-family: 'CustomFont', sans-serif;
}

:deep(.el-button) {
  font-family: 'CustomFont', sans-serif;
}
</style>