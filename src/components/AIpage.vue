<template>
  <div class="chat-layout">
    <!-- 侧边栏 -->
    <aside class="sidebar">
      <button class="new-chat-btn" @click="startNewChat">+ 新对话</button>
      <div class="conv-list">
        <div
          v-for="conv in conversations"
          :key="conv.id"
          :class="['conv-item', { active: conv.id === conversationId }]"
          @click="loadConversation(conv.id)"
        >
          <span class="conv-title">{{ conv.title }}</span>
          <button class="del-btn" @click.stop="deleteConversation(conv.id)">×</button>
        </div>
      </div>
    </aside>

    <!-- 主聊天区 -->
    <div class="chat-container">
      <h1>🤖 我的第一个 AI 助手</h1>
      <p class="subtitle">Powered by Cloudflare AI + Vue 3 + D1 持久化</p>
      
      <div class="chat-box" ref="chatBox">
        <div v-for="(msg, idx) in messages" :key="idx" :class="['message', msg.role]">
          <div class="avatar">{{ msg.role === 'user' ? '👤' : '🤖' }}</div>
          <div class="content">{{ msg.content }}</div>
          <div class="message-actions">
            <button v-if="msg.role === 'assistant'" @click="regenerate" title="重新生成">🔄</button>
            <button v-if="msg.role === 'user'" @click="editMessage(msg)" title="编辑">✏️</button>
          </div>
        </div>
        <div v-if="isLoading" class="message assistant">
          <div class="avatar">🤖</div>
          <div class="content typing">正在思考中...</div>
        </div>
      </div>

      <div class="token-info">
        📊 上下文用量: {{ estimatedTokens }} / {{ MAX_TOKENS }} tokens
        <span v-if="estimatedTokens > MAX_TOKENS * 0.8" style="color: orange;">⚠️ 接近上限</span>
      </div>

      <div class="input-area">
        <input 
          v-model="inputText" 
          @keydown.enter="sendMessage" 
          placeholder="输入消息，按 Enter 发送..."
          :disabled="isLoading || !conversationId"
        />
        <button @click="sendMessage" :disabled="isLoading || !inputText.trim() || !conversationId">发送</button>
        <button @click="stopGeneration" :disabled="!isLoading">停止</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted } from 'vue';

const API_URL = 'https://chat.chusxxin.cyou';

const SYSTEM_PROMPT = { role: 'system', content: '你是 chusxxin 的私人助手，回答简洁友好。' };

// ---- 会话管理 ----
const conversationId = ref(null);
const conversations = ref([]);

// ---- 聊天状态 ----
const chatHistory = ref([{ ...SYSTEM_PROMPT }]);
const messages = ref([]);
const inputText = ref('');
const isLoading = ref(false);
const estimatedTokens = ref(0);
const MAX_TOKENS = 8000;

const abortController = ref(null);
const chatBox = ref(null);

// ================= 辅助函数 =================
const scrollToBottom = async () => {
  await nextTick();
  if (chatBox.value) {
    chatBox.value.scrollTop = chatBox.value.scrollHeight;
  }
};

const updateTokenEstimate = () => {
  const allText = messages.value.map(m => m.content).join(' ');
  estimatedTokens.value = Math.ceil(allText.length / 2.5);
};

// ================= 会话操作 =================
const fetchConversations = async () => {
  try {
    const res = await fetch(`${API_URL}/conversations`);
    conversations.value = await res.json();
  } catch (e) {
    console.error('获取会话列表失败', e);
  }
};

const startNewChat = async () => {
  try {
    const res = await fetch(`${API_URL}/conversations`, { method: 'POST' });
    const data = await res.json();
    conversationId.value = data.id;
    messages.value = [];
    chatHistory.value = [{ ...SYSTEM_PROMPT }];
    estimatedTokens.value = 0;
    inputText.value = '';
    await fetchConversations(); // 刷新侧边栏
  } catch (e) {
    console.error('创建新会话失败', e);
  }
};

const loadConversation = async (id) => {
  try {
    const res = await fetch(`${API_URL}/conversation?id=${id}`);
    const history = await res.json(); // [{role, content}, ...]
    conversationId.value = id;
    messages.value = history.map(m => ({ role: m.role, content: m.content }));
    chatHistory.value = [
      { ...SYSTEM_PROMPT },
      ...history.map(m => ({ role: m.role, content: m.content }))
    ];
    updateTokenEstimate();
    await scrollToBottom();
  } catch (e) {
    console.error('加载对话失败', e);
  }
};

const deleteConversation = async (id) => {
  if (!confirm('确定要删除这个对话吗？')) return;
  try {
    await fetch(`${API_URL}/conversation?id=${id}`, { method: 'DELETE' });
    // 如果删除的是当前打开的会话，则清空界面
    if (conversationId.value === id) {
      conversationId.value = null;
      messages.value = [];
      chatHistory.value = [{ ...SYSTEM_PROMPT }];
      estimatedTokens.value = 0;
    }
    await fetchConversations(); // 刷新列表
  } catch (e) {
    console.error('删除会话失败', e);
  }
};

// ================= 交互按钮 =================
const stopGeneration = () => {
  if (abortController.value) {
    abortController.value.abort();
  }
};

const regenerate = async () => {
  if (isLoading.value) return;
  const lastUserMsg = messages.value.findLast(msg => msg.role === 'user');
  if (!lastUserMsg) return;

  const lastAssistantIndex = messages.value.findLastIndex(msg => msg.role === 'assistant');
  if (lastAssistantIndex !== -1) {
    messages.value.splice(lastAssistantIndex, 1);
    const historyLastAssistant = chatHistory.value.findLastIndex(m => m.role === 'assistant');
    if (historyLastAssistant !== -1) chatHistory.value.splice(historyLastAssistant, 1);
  }

  inputText.value = lastUserMsg.content;
  await sendMessage();
};

const editMessage = (msg) => {
  if (isLoading.value) return;
  inputText.value = msg.content;
  const index = messages.value.findIndex(m => m === msg);
  if (index !== -1) {
    messages.value.splice(index);
    const historyIndex = chatHistory.value.findIndex(m => m.role === 'user' && m.content === msg.content);
    if (historyIndex !== -1) {
      chatHistory.value.splice(historyIndex);
    }
  }
  document.querySelector('input')?.focus();
};

// ================= 发送消息 =================
const sendMessage = async () => {
  const text = inputText.value.trim();
  if (!text || isLoading.value || !conversationId.value) return;

  if (abortController.value) {
    abortController.value.abort();
  }
  abortController.value = new AbortController();

  const userMessage = { role: 'user', content: text };
  messages.value.push(userMessage);
  chatHistory.value.push(userMessage);
  
  inputText.value = '';
  await scrollToBottom();
  isLoading.value = true;

  try {
    const response = await fetch(`${API_URL}/chat`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        messages: chatHistory.value,
        conversationId: conversationId.value
      }),
      signal: abortController.value.signal,
    });

    const contentType = response.headers.get('content-type');
    if (contentType && contentType.includes('application/json')) {
      const errorData = await response.json();
      messages.value.push({ role: 'assistant', content: `后端错误: ${errorData.error}` });
      return;
    }

    if (!response.body) throw new Error('ReadableStream not supported');

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    messages.value.push({ role: 'assistant', content: '' });
    const aiMessageIndex = messages.value.length - 1;

    let buffer = '';
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6).trim();
          if (data === '[DONE]') continue;
          try {
            const parsed = JSON.parse(data);
            if (parsed.response) {
              messages.value[aiMessageIndex].content += parsed.response;
              await scrollToBottom();
            }
          } catch (e) {}
        }
      }
    }

    const aiContent = messages.value[aiMessageIndex].content;
    chatHistory.value.push({ role: 'assistant', content: aiContent });
    updateTokenEstimate();

  } catch (error) {
    if (error.name === 'AbortError') {
      messages.value.push({ role: 'assistant', content: '[已停止生成]' });
    } else {
      console.error('请求失败:', error);
      messages.value.push({ role: 'assistant', content: '网络错误，请检查你的 API 是否在线。' });
    }
  } finally {
    isLoading.value = false;
    abortController.value = null;
    await scrollToBottom();
  }
};

// ================= 入口 =================
onMounted(async () => {
  await fetchConversations(); // 只拉列表，不自动创建新会话
});
</script>

<style scoped>

* {
  box-sizing: border-box;
}
/* 移除全局 margin:0，改用具体元素定义 */
body, h1, p, input, button {
  margin: 0;
  padding: 0;
}

/* ---- 整体布局 ---- */
.chat-layout {
  display: flex;
  height: 100vh;
  background: #fafafa;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Helvetica Neue', sans-serif;
}

/* ---- 侧边栏 ---- */
.sidebar {
  width: 260px;
  background: #ffffff;
  border-right: 1px solid #f0f0f2;
  padding: 20px 16px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  overflow-y: auto;
}

.new-chat-btn {
  background: #18181b;
  color: white;
  border: none;
  border-radius: 30px;
  padding: 10px 18px;
  font-size: 0.9rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
  text-align: center;
}

.new-chat-btn:hover {
  background: #2c2c30;
  transform: scale(1.02);
}

.conv-list {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.conv-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 12px;
  border-radius: 12px;
  cursor: pointer;
  font-size: 0.85rem;
  color: #4b5563;
  transition: background 0.15s;
}

.conv-item:hover {
  background: #f3f4f6;
}

.conv-item.active {
  background: #f3f4f6;
  color: #171717;
  font-weight: 500;
}

.conv-title {
  flex: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-right: 8px;
}

.del-btn {
  background: none;
  border: none;
  font-size: 1.1rem;
  color: #9ca3af;
  cursor: pointer;
  padding: 0 4px;
  border-radius: 4px;
  transition: color 0.1s;
}

.del-btn:hover {
  color: #ef4444;
}


/* ---- 主聊天区域 ---- */
.chat-container {
  flex: 1;
  max-width: 860px;
  margin: 0 auto;
  padding: 24px 20px;
  display: flex;
  flex-direction: column;
  background: #fafafa;
}

h1 {
  text-align: center;
  color: #171717;
  font-weight: 600;
  font-size: 1.8rem;
  letter-spacing: -0.02em;
  margin-bottom: 4px;
}

.subtitle {
  text-align: center;
  color: #6b7280;
  font-size: 0.9rem;
  font-weight: 400;
  margin-top: 0;
  margin-bottom: 24px;
}

.chat-box {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  background: #ffffff;
  border-radius: 24px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.02), 0 1px 3px rgba(0, 0, 0, 0.03);
  border: 1px solid rgba(0, 0, 0, 0.03);
  margin-bottom: 20px;
}

/* 自定义滚动条 */
.chat-box::-webkit-scrollbar {
  width: 6px;
}
.chat-box::-webkit-scrollbar-track {
  background: transparent;
}
.chat-box::-webkit-scrollbar-thumb {
  background: #d1d5db;
  border-radius: 12px;
}

/* 消息气泡 */
.message {
  display: flex;
  margin-bottom: 24px;
  gap: 12px;
}
.message.user {
  flex-direction: row-reverse;
}
.avatar {
  width: 34px;
  height: 34px;
  border-radius: 50%;
  background: #f3f4f6;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  box-shadow: 0 1px 2px rgba(0,0,0,0.02);
  flex-shrink: 0;
}
.message.user .avatar {
  background: #18181b;
  color: #fafafa;
  box-shadow: 0 2px 6px rgba(0,0,0,0.08);
}
.content {
  background: #ffffff;
  padding: 14px 18px;
  border-radius: 20px;
  max-width: 75%;
  word-break: break-word;
  font-size: 0.95rem;
  line-height: 1.55;
  color: #1f2937;
  box-shadow: 0 1px 3px rgba(0,0,0,0.02);
  border: 1px solid #f0f0f2;
}
.message.user .content {
  background: #18181b;
  color: #ffffff;
  border: none;
  box-shadow: 0 4px 8px -2px rgba(0,0,0,0.08);
}
.typing {
  color: #9ca3af;
  font-style: normal;
  display: flex;
  align-items: center;
  gap: 4px;
}
.typing::after {
  content: '';
  width: 6px;
  height: 6px;
  background: #9ca3af;
  border-radius: 50%;
  display: inline-block;
  animation: pulse 1.2s infinite;
}
@keyframes pulse {
  0%, 100% { opacity: 0.3; transform: scale(0.9); }
  50% { opacity: 1; transform: scale(1); }
}

/* 消息操作按钮 */
.message-actions {
  display: flex;
  gap: 8px;
  margin-left: 8px;
  opacity: 0;
  transition: opacity 0.2s ease;
  align-self: flex-end;
}
.message:hover .message-actions {
  opacity: 1;
}
.message-actions button {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 30px;
  padding: 6px 12px;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  color: #4b5563;
  box-shadow: 0 2px 4px rgba(0,0,0,0.02);
  transition: all 0.15s;
  display: inline-flex;
  align-items: center;
  gap: 4px;
}
.message-actions button:hover {
  background: #f9fafb;
  border-color: #18181b;
  color: #18181b;
  box-shadow: 0 4px 8px -2px rgba(0,0,0,0.06);
}

/* Token 用量 */
.token-info {
  text-align: right;
  font-size: 0.8rem;
  color: #9ca3af;
  margin-bottom: 10px;
  padding-right: 8px;
  font-feature-settings: "tnum";
  font-variant-numeric: tabular-nums;
}

/* 输入区 */
.input-area {
  display: flex;
  gap: 12px;
  background: #ffffff;
  padding: 8px 8px 8px 20px;
  border-radius: 60px;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.02), 0 0 0 1px rgba(0, 0, 0, 0.01);
  border: 1px solid #f0f0f2;
  transition: box-shadow 0.2s;
}
.input-area:focus-within {
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.04), 0 0 0 2px rgba(24, 24, 27, 0.05);
  border-color: transparent;
}
input {
  flex: 1;
  border: none;
  outline: none;
  font-size: 16px;
  padding: 12px 0;
  background: transparent;
  color: #171717;
}
input::placeholder {
  color: #cbd5e1;
  font-weight: 350;
}
button {
  background: #18181b;
  color: white;
  border: none;
  border-radius: 40px;
  padding: 10px 22px;
  font-size: 0.95rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.04);
  letter-spacing: -0.01em;
}
button:hover:not(:disabled) {
  background: #2c2c30;
  box-shadow: 0 6px 12px -2px rgba(0, 0, 0, 0.08);
  transform: scale(1.02);
}
button:active:not(:disabled) {
  transform: scale(0.98);
}
button:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  box-shadow: none;
  transform: none;
}
.input-area button:last-of-type {
  background: #ffffff;
  color: #4b5563;
  border: 1px solid #e5e7eb;
  box-shadow: none;
  padding: 10px 20px;
}
.input-area button:last-of-type:hover:not(:disabled) {
  background: #f3f4f6;
  border-color: #d1d5db;
  color: #18181b;
}

/* 移动端适配 */
@media (max-width: 700px) {
  /* 隐藏侧边栏，让主区域占满全屏 */
  .sidebar {
    display: none;
  }

  /* 主布局改为纯聊天区，取消 flex 布局 */
  .chat-layout {
    display: block;
    height: 100vh;
    padding: 0;
  }

  /* 主聊天区域全宽，减少内边距 */
  .chat-container {
    max-width: 100%;
    margin: 0;
    padding: 12px 10px;
    height: 100vh;
    display: flex;
    flex-direction: column;
  }

  /* 标题和副标题缩小 */
  h1 {
    font-size: 1.4rem;
    margin-bottom: 2px;
  }
  .subtitle {
    font-size: 0.75rem;
    margin-bottom: 12px;
  }

  /* 聊天消息区域自动填充剩余空间 */
  .chat-box {
    flex: 1;
    padding: 12px;
    border-radius: 16px;
    margin-bottom: 10px;
  }

  /* 消息气泡宽度更宽，充分利用空间 */
  .content {
    max-width: 85%;
    padding: 10px 14px;
    font-size: 0.9rem;
  }
  .message.user .content {
    max-width: 85%;
  }

  /* 确保头像在小屏幕上不被挤压 */
  .avatar {
    width: 30px;
    height: 30px;
    font-size: 16px;
  }

  /* Token 信息缩小 */
  .token-info {
    font-size: 0.7rem;
    margin-bottom: 6px;
  }

  /* 输入区域移动端优化 */
  .input-area {
    padding: 6px 8px 6px 14px;
    border-radius: 40px;
    gap: 6px;
    box-shadow: 0 2px 10px rgba(0,0,0,0.04);
  }

  /* 输入框 */
  .input-area input {
    flex: 1;
    font-size: 16px;          /* 禁止 iOS 自动缩放 */
    padding: 10px 0;
    min-width: 0;
  }

  /* 所有按钮增大触控区域 */
  button {
    padding: 10px 16px;
    font-size: 0.9rem;
    min-height: 44px;         /* 满足 WCAG 触控尺寸 */
    min-width: 44px;
  }

  /* 发送按钮保持主色调 */
  .input-area button:not(:last-child) {
    padding: 8px 16px;
  }

  /* 停止按钮的次要样式也保持 */
  .input-area button:last-of-type {
    padding: 8px 14px;
  }

  /* 消息操作按钮保持可用 */
  .message-actions button {
    padding: 4px 10px;
    font-size: 12px;
  }
}
</style>