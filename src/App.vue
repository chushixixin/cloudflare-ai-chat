<template>
  <div>
    <nav class="top-nav">
      <button @click="currentView = 'chat'" :class="{ active: currentView === 'chat' }">💬 AI 对话</button>
      <button @click="currentView = 'monitor'" :class="{ active: currentView === 'monitor' }">📊 情报监控</button>
    </nav>
    <AIpage v-if="currentView === 'chat'" />
    <MonitorPage v-if="currentView === 'monitor'" />
  </div>
</template>

<script setup>
import { ref } from 'vue';
import AIpage from './components/AIpage.vue';
import MonitorPage from './components/MonitorPage.vue';

const currentView = ref('chat');
</script>

<style>
.top-nav {
  display: flex;
  justify-content: center;
  gap: 16px;
  padding: 16px;
  background: #fff;
  border-bottom: 1px solid #f0f0f2;
}
.top-nav button {
  background: transparent;
  border: none;
  font-size: 1rem;
  padding: 8px 18px;
  border-radius: 30px;
  cursor: pointer;
  color: #4b5563;
  transition: all 0.15s;
}
.top-nav button.active {
  background: #18181b;
  color: #fff;
}
</style>