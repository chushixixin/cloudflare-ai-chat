<template>
  <div class="chat-container">
    <h1>🤖 我的第一个 AI 助手</h1>
    <p class="subtitle">Powered by Cloudflare AI + Vue 3</p>
    
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
        :disabled="isLoading"
      />
      <button @click="sendMessage" :disabled="isLoading || !inputText.trim()">发送</button>
      <button @click="stopGeneration" :disabled="!isLoading">停止</button>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue';

const API_URL = 'https://chat.chusxxin.cyou';

const chatHistory = ref([
  { role: 'system', content: '你是 chusxxin 的私人助手，回答简洁友好。' }
]);

const abortController = ref(null);
const inputText = ref('');
const messages = ref([]);
const isLoading = ref(false);
const chatBox = ref(null);
const estimatedTokens = ref(0);
const MAX_TOKENS = 8000;

const scrollToBottom = async () => {
  await nextTick();
  if (chatBox.value) {
    chatBox.value.scrollTop = chatBox.value.scrollHeight;
  }
};

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

const updateTokenEstimate = () => {
  const allText = messages.value.map(m => m.content).join(' ');
  estimatedTokens.value = Math.ceil(allText.length / 2.5);
};

const sendMessage = async () => {
  const text = inputText.value.trim();
  if (!text || isLoading.value) return;

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
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ messages: chatHistory.value }),
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
</script>

<style scoped>
* {
  box-sizing: border-box;
  margin: 0;
}

.chat-container {
  max-width: 880px;
  margin: 0 auto;
  padding: 24px 20px;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #fafafa;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Helvetica Neue', sans-serif;
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
  scroll-behavior: smooth;
}

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
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.02);
  flex-shrink: 0;
  transition: transform 0.1s ease;
}

.message.user .avatar {
  background: #18181b;
  color: #fafafa;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.08);
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
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.02);
  border: 1px solid #f0f0f2;
}

.message.user .content {
  background: #18181b;
  color: #ffffff;
  border: none;
  box-shadow: 0 4px 8px -2px rgba(0, 0, 0, 0.08);
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

.token-info {
  text-align: right;
  font-size: 0.8rem;
  color: #9ca3af;
  margin-bottom: 10px;
  padding-right: 8px;
  font-feature-settings: "tnum";
  font-variant-numeric: tabular-nums;
}

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
  font-size: 0.95rem;
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

@media (max-width: 600px) {
  .chat-container {
    padding: 12px;
  }
  
  .content {
    max-width: 85%;
    padding: 12px 14px;
  }
  
  .input-area {
    padding: 6px 6px 6px 16px;
  }
  
  button {
    padding: 8px 16px;
  }
}
</style>