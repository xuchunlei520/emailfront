<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import CopyIcon from './assets/icon/复制.svg'
import UpdateIcon from './assets/icon/更新.svg'

// API基础URL
const API_BASE_URL = '/api'

// 临时邮箱地址
const tempEmail = ref('')
const currentEmailData = ref(null)
const emails = ref([])
const loading = ref(false)
const selectedEmailId = ref(null)
// 复制状态
const isCopied = ref(false)
// 生成新的临时邮箱地址
const generateNewEmail = async () => {
  try {
    loading.value = true
    const response = await fetch(`${API_BASE_URL}/email/generate`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({})
    })
    
    if (!response.ok) {
      throw new Error('生成邮箱失败')
    }
    
    const result = await response.json()
    if (result.success) {
      tempEmail.value = result.data.address
      currentEmailData.value = result.data
      emails.value = [] // 清空之前的邮件
      // console.log('新邮箱生成成功:', result.data)
    }
  } catch (error) {
    console.error('生成邮箱失败:', error)
    alert('生成邮箱失败，请重试')
  } finally {
    loading.value = false
  }
}

// 复制邮箱地址
const copyEmail = async () => {
  try {
    await navigator.clipboard.writeText(tempEmail.value)
    isCopied.value = true
  } catch (err) {
    console.error('复制失败:', err)
    // 降级方案
    const textArea = document.createElement('textarea')
    textArea.value = tempEmail.value
    document.body.appendChild(textArea)
    textArea.select()
    document.execCommand('copy')
    document.body.removeChild(textArea)
    isCopied.value = true
  }
}

// 刷新收件箱
const refreshInbox = async (silent = false) => {
  if (!tempEmail.value) return
  
  try {
    if (!silent) {
      loading.value = true
    }
    const response = await fetch(`${API_BASE_URL}/email/emails/${tempEmail.value}`)
    
    if (!response.ok) {
      throw new Error('获取邮件失败')
    }
    
    const result = await response.json()
    if (result.success) {
      const newEmails = result.data.emails || []
      const oldEmailIds = new Set(emails.value.map(email => email.id))
      const newEmailIds = new Set(newEmails.map(email => email.id))
      
      // 检查是否有新邮件
      const hasNewEmails = newEmails.length > emails.value.length || 
                          newEmails.some(email => !oldEmailIds.has(email.id))
      
      // 更新邮件列表
      emails.value = newEmails
      
      if (hasNewEmails) {
        // console.log('发现新邮件，界面已更新')
        // 如果有新邮件且当前没有展开的邮件详情，可以显示提示
        if (!selectedEmailId.value && newEmails.length > 0) {
          // console.log('新邮件:', newEmails.filter(email => !oldEmailIds.has(email.id)))
        }
      } else if (!silent) {
        // console.log('邮件列表更新（无新邮件）:', result.data)
      }
    }
  } catch (error) {
    console.error('获取邮件失败:', error)
  } finally {
    if (!silent) {
      loading.value = false
    }
  }
}

// 删除邮箱
const deleteEmail = () => {
  generateNewEmail()
}

// 获取邮件详情
const getEmailDetail = (emailId) => {
  // 切换选中状态：如果已选中则收起，未选中则展开
  selectedEmailId.value = selectedEmailId.value === emailId ? null : emailId
}

// 删除单个邮件
const deleteSingleEmail = async (emailId) => {
  if (!tempEmail.value || !emailId) {
    console.warn('缺少邮箱地址或邮件ID')
    return
  }
  try {
    const response = await fetch(`${API_BASE_URL}/email/emails/${tempEmail.value}/${emailId}`, {
      method: 'DELETE'
    })
    
    if (!response.ok) {
      throw new Error('删除邮件失败')
    }
    
    const result = await response.json()
    if (result.success) {
      // 刷新邮件列表
      await refreshInbox()
      // console.log('邮件删除成功')
    }
  } catch (error) {
    console.error('删除邮件失败:', error)
  }
}

// 定时器引用
let refreshTimer = null

// 组件挂载时生成初始邮箱
onMounted(() => {
  generateNewEmail()
  
  // 设置定时静默刷新收件箱（每10秒）
  refreshTimer = setInterval(() => {
    if (tempEmail.value) {
      refreshInbox(true) // 静默刷新，不显示loading状态
    }
  }, 10000)
})

// 组件卸载时清理定时器
onUnmounted(() => {
  if (refreshTimer) {
    clearInterval(refreshTimer)
    refreshTimer = null
  }
})
</script>

<template>
  <div class="app">
    <!-- 顶部导航栏 -->
    <header class="header">
      <div class="header-content">
        <!-- <div class="header-left">
          <a href="#" class="store-link">
            <span class="apple-logo">🍎</span>
            App Store
          </a>
          <a href="#" class="store-link">
            <span class="google-logo">📱</span>
            Google Play
          </a>
        </div> -->
        
        <div class="header-center">
          <div class="logo">
            <span class="checkmark">✓</span>
            Qfun临时免费邮件
          </div>
        </div>
        
        <!-- <div class="header-right">
          <button class="refresh-btn" @click="refreshInbox(false)" :disabled="loading" title="刷新收件箱">
            <img :src="UpdateIcon" alt="刷新" :class="{ 'rotating-icon': loading }" class="refresh-icon" />
            {{ loading ? '刷新中...' : '刷新' }}
          </button>
        </div> -->
        
        <!-- <div class="header-right">
          <button class="temp-number-btn">
            <span class="document-icon">📄</span>
            Temp Number
          </button>
          <button class="premium-btn">Premium</button>
        </div> -->
      </div>
    </header>

    <!-- 主要内容区域 -->
    <main class="main-content">
      <div class="email-section">
        <div class="email-container">
          <h1 class="email-title">你的临时电子邮件地址</h1>
          <div class="email-input-container">
            <input 
              type="text" 
              :value="tempEmail" 
              readonly 
              class="email-input"
            />
            <div class="email-actions">
              <button class="copy-btn" @click="copyEmail" title="复制邮箱地址">
                <img :src="CopyIcon" alt="复制" class="copy-icon" />{{ isCopied ? '已复制' : '' }}
              </button>
              <button class="change-btn" @click="generateNewEmail" title="换一个邮箱地址">
                <span class="change-icon">换一个</span>
              </button>
            </div>
          </div>
        </div>
        
        <p class="description">
          不用再担心垃圾邮件,广告邮件,黑客和机器人攻击。让您真实的邮箱保持干净和安全。Temp Mail提供临时、安全、匿名、免费的一次性电子邮件地址。
        </p>
      </div>

      <!-- 操作按钮区域 -->
      <!-- <div class="action-buttons">
        <button class="action-btn" @click="copyEmail">
          <span class="action-icon">📋</span>
          拷贝
        </button>
        <button class="action-btn" @click="refreshInbox">
          <span class="action-icon">🔄</span>
          刷新
        </button>
        <button class="action-btn" @click="generateNewEmail">
          <span class="action-icon">✏️</span>
          变更
        </button>
        <button class="action-btn" @click="deleteEmail">
          <span class="action-icon">❌</span>
          删除
        </button>
      </div> -->

      <!-- 收件箱区域 -->
      <div class="inbox-container">
        <div class="inbox-section">
          <div class="inbox-header">
            <div class="inbox-header-item">发送人</div>
            <div class="inbox-header-item">主题:</div>
            <div class="inbox-header-item">浏览</div>
          </div>
          
          <div class="inbox-content">
            <!-- 加载状态 -->
            <div v-if="loading" class="loading-inbox">
              <div class="empty-icon">
                <img :src="UpdateIcon" alt="更新" class="rotating-icon" />
              </div>
              <div class="empty-text">
                <div>正在加载...</div>
              </div>
            </div>
            
            <!-- 空收件箱 -->
            <div v-else-if="emails.length === 0" class="empty-inbox">
              <div class="empty-icon">
                <img :src="UpdateIcon" alt="更新" class="rotating-icon" />
              </div>
              <div class="empty-text">
                <div>您的收件箱是空的</div>
                <div>正在等待来信</div>
              </div>
            </div>
            
            <!-- 邮件列表 -->
            <div v-else class="email-list">
              <div v-for="email in emails" :key="email.id" class="email-item-wrapper">
                <div 
                  class="email-item"
                  :class="{selected: selectedEmailId === email.id}"
                  @click="getEmailDetail(email.id)"
                >
                  <div class="email-from">{{ email.from }}</div>
                  <div class="email-subject">{{ email.subject || '无主题' }}</div>
                  <div class="email-actions">
                    <button 
                      class="email-action-btn" 
                      @click.stop="getEmailDetail(email.id)"
                      title="查看详情"
                    >
                      {{ selectedEmailId === email.id ? '🔽' : '👁️' }}
                    </button>
                    <button 
                      class="email-action-btn delete-btn" 
                      @click.stop="deleteSingleEmail(email.id)"
                      title="删除邮件"
                    >
                      🗑️
                    </button>
                  </div>
                </div>
                
                <!-- 邮件详情展开区域 -->
                <div v-if="selectedEmailId === email.id" class="email-detail">
                  <div class="email-detail-header">
                    <h3>{{ email.subject || '无主题' }}</h3>
                    <div class="email-detail-meta">
                      <div class="meta-item">
                        <strong>发件人:</strong> {{ email.from }}
                      </div>
                      <div class="meta-item">
                        <strong>收件人:</strong> {{ email.to }}
                      </div>
                      <div class="meta-item">
                        <strong>时间:</strong> {{ email.createdAt ? new Date(email.createdAt).toLocaleString() : '未知时间' }}
                      </div>
                    </div>
                  </div>
                  <div class="email-detail-content">
                    <div v-if="email.html" v-html="email.html" class="email-html-content"></div>
                    <div v-else-if="email.text" class="email-text-content">{{ email.text }}</div>
                    <div v-else class="email-no-content">无内容</div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

/* 顶部导航栏样式 */
.header {
  background-color: #2c2c2c;
  padding: 0 20px;
  border-bottom: 1px solid #404040;
}

.header-content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  max-width: 1200px;
  margin: 0 auto;
  height: 60px;
}

.header-left {
  display: flex;
  gap: 20px;
}

.store-link {
  color: white;
  text-decoration: none;
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 14px;
  transition: opacity 0.2s;
}

.store-link:hover {
  opacity: 0.8;
}

.header-center {
  flex: 1;
  display: flex;
  justify-content: center;
}

.logo {
  color: white;
  font-size: 24px;
  font-weight: bold;
  display: flex;
  align-items: center;
  gap: 8px;
}

.checkmark {
  color: #4CAF50;
  font-size: 20px;
}

.header-right {
  display: flex;
  gap: 15px;
  align-items: center;
}

.refresh-btn {
  background: linear-gradient(145deg, #4CAF50, #45a049);
  border: 1px solid #5cb85c;
  color: white;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 8px;
  transition: all 0.3s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.refresh-btn:hover:not(:disabled) {
  background: linear-gradient(145deg, #5cb85c, #4CAF50);
  border-color: #66bb6a;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.refresh-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none;
}

.refresh-icon {
  width: 16px;
  height: 16px;
  filter: invert(1);
}

.refresh-icon.rotating-icon {
  animation: rotate 1s linear infinite;
}

.temp-number-btn, .premium-btn {
  background: none;
  border: none;
  color: white;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  display: flex;
  align-items: center;
  gap: 5px;
  transition: background-color 0.2s;
}

.temp-number-btn:hover {
  background-color: #404040;
}

.premium-btn {
  background-color: #FFD700;
  color: #333;
  font-weight: bold;
}

.premium-btn:hover {
  background-color: #FFC107;
}

/* 主要内容区域 */
.main-content {
  background-color: #2c2c2c;
  min-height: calc(100vh - 60px);
}

/* 收件箱容器 */
.inbox-container {
  background-color: #f6f7f9;
  padding: 40px 20px;
  min-height: 500px;
}

/* 收件箱区域背景 */
.inbox-section {
  background-color: white;
  min-height: 400px;
  max-width: 800px;
  margin: 0 auto;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.email-section {
  padding: 60px 20px 40px;
  max-width: 800px;
  margin: 0 auto;
  text-align: center;
}

.email-container {
  border: 2px dashed #666;
  border-radius: 12px;
  padding: 40px 30px;
  margin-bottom: 30px;
  background-color: #333;
}

.email-title {
  color: white;
  font-size: 28px;
  font-weight: bold;
  margin: 0 0 30px 0;
}

.email-input-container {
  display: flex;
  align-items: center;
  gap: 15px;
  justify-content: center;
  flex-wrap: wrap;
}

.email-input {
  background-color: #444;
  border: 2px solid #666;
  border-radius: 8px;
  padding: 15px 20px;
  color: white;
  font-size: 18px;
  min-width: 300px;
  text-align: center;
  outline: none;
}

.email-input:focus {
  border-color: #4CAF50;
}

.email-actions {
  display: flex;
  gap: 10px;
}

.copy-btn, .change-btn {
  background: linear-gradient(145deg, #3a3a3a, #2a2a2a);
  border: 1px solid #555;
  color: white;
  padding: 12px 16px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.3s ease;
  box-shadow: 
    0 2px 4px rgba(0, 0, 0, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 80px;
  justify-content: center;
}

.copy-btn:hover, .change-btn:hover {
  background: linear-gradient(145deg, #4a4a4a, #3a3a3a);
  border-color: #666;
  transform: translateY(-1px);
  box-shadow: 
    0 4px 8px rgba(0, 0, 0, 0.4),
    inset 0 1px 0 rgba(255, 255, 255, 0.2);
}

.copy-btn:active, .change-btn:active {
  transform: translateY(0);
  box-shadow: 
    0 1px 2px rgba(0, 0, 0, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
}

.copy-btn {
  background: linear-gradient(145deg, #4CAF50, #45a049);
  border-color: #5cb85c;
}

.copy-btn:hover {
  background: linear-gradient(145deg, #5cb85c, #4CAF50);
  border-color: #66bb6a;
}

.copy-icon {
  width: 18px;
  height: 18px;
  filter: invert(1);
}

.change-icon {
  font-size: 16px;
  font-weight: 600;
}

.description {
  color: white;
  font-size: 16px;
  line-height: 1.6;
  margin: 0;
  opacity: 0.9;
}

/* 操作按钮区域 */
.action-buttons {
  display: flex;
  justify-content: center;
  gap: 20px;
  padding: 20px;
  background-color: #f5f5f5;
  border-top: 1px solid #ddd;
}

.action-btn {
  background-color: #e0e0e0;
  border: none;
  border-radius: 8px;
  padding: 12px 20px;
  color: #333;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 8px;
  transition: background-color 0.2s;
}

.action-btn:hover {
  background-color: #d0d0d0;
}

.action-icon {
  font-size: 16px;
}

/* 收件箱区域 */
.inbox-section {
  background-color: white;
  min-height: 400px;
  max-width: 800px;
  margin: 10px auto;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.inbox-header {
  background-color: #2c2c2c;
  color: white;
  display: flex;
  padding: 15px 20px;
  font-weight: 500;
  border-radius: 12px 12px 0 0;
}

.inbox-header-item {
  flex: 1;
  text-align: left;
}

.inbox-header-item:nth-child(2) {
  text-align: center;
}

.inbox-header-item:nth-child(3) {
  text-align: right;
}

.inbox-content {
  padding: 60px 20px;
  display: flex;
  justify-content: center;
  align-items: center;
}

.empty-inbox {
  text-align: center;
  color: #666;
}

.empty-icon {
  margin-bottom: 20px;
  opacity: 0.5;
  display: flex;
  justify-content: center;
  align-items: center;
}

.rotating-icon {
  width: 80px;
  height: 80px;
  animation: rotate 2s linear infinite;
}

@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.empty-text {
  font-size: 18px;
  line-height: 1.5;
}

.empty-text div:first-child {
  font-weight: 500;
  margin-bottom: 5px;
}

/* 邮件列表样式 */
.email-list {
  width: 100%;
}

.email-item-wrapper {
  border-bottom: 1px solid #eee;
}

.email-item-wrapper:last-child {
  border-bottom: none;
}

.email-item {
  display: flex;
  align-items: center;
  padding: 15px 20px;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
}

.email-item:hover {
  background-color: #f8f9fa;
}

.email-item.selected {
  background-color: #e3f2fd;
}



.email-from {
  flex: 1;
  font-weight: 500;
  color: #333;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  margin-right: 15px;
}

.email-subject {
  flex: 2;
  color: #666;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  margin-right: 15px;
}

.email-actions {
  display: flex;
  gap: 8px;
}

.email-action-btn {
  background: none;
  border: none;
  padding: 8px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.2s;
}

.email-action-btn:hover {
  background-color: #e9ecef;
}

.delete-btn:hover {
  background-color: #f8d7da;
}

/* 邮件详情展开区域 */
.email-detail {
  background-color: #f8f9fa;
  border-top: 1px solid #e9ecef;
  animation: slideDown 0.3s ease-out;
}

@keyframes slideDown {
  from {
    opacity: 0;
    max-height: 0;
    padding: 0 20px;
  }
  to {
    opacity: 1;
    max-height: 500px;
    padding: 20px;
  }
}

.email-detail-header {
  margin-bottom: 15px;
}

.email-detail-header h3 {
  margin: 0 0 10px 0;
  color: #333;
  font-size: 18px;
  font-weight: 600;
}

.email-detail-meta {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.meta-item {
  font-size: 14px;
  color: #666;
  line-height: 1.4;
}

.meta-item strong {
  color: #333;
  font-weight: 600;
}

.email-detail-content {
  background-color: white;
  border: 1px solid #e9ecef;
  border-radius: 6px;
  padding: 15px;
  margin-top: 10px;
}

.email-html-content {
  line-height: 1.6;
  color: #333;
  word-wrap: break-word;
  overflow-wrap: break-word;
}

.email-html-content img {
  max-width: 100%;
  height: auto;
}

.email-html-content a {
  color: #2196f3;
  text-decoration: none;
}

.email-html-content a:hover {
  text-decoration: underline;
}

.email-text-content {
  line-height: 1.6;
  color: #333;
  white-space: pre-wrap;
  word-wrap: break-word;
  overflow-wrap: break-word;
}

.email-no-content {
  color: #999;
  font-style: italic;
  text-align: center;
  padding: 20px;
}

/* 加载状态 */
.loading-inbox {
  text-align: center;
  color: #666;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .header-content {
    flex-direction: column;
    height: auto;
    padding: 15px 0;
    gap: 15px;
  }
  
  .header-left, .header-right {
    justify-content: center;
  }
  
  .email-input-container {
    flex-direction: column;
    align-items: center;
  }
  
  .email-input {
    min-width: 250px;
  }
  
  .action-buttons {
    flex-wrap: wrap;
    gap: 10px;
  }
  
  .action-btn {
    flex: 1;
    min-width: 120px;
    justify-content: center;
  }
}
</style>
