<template>
  <div class="monitor-container">
    <h1>🤖 情报监控中心</h1>
    <p class="subtitle">AI 自动抓取 · 智能摘要 · 每日更新</p>

    <div class="controls">
      <button @click="triggerFetch" :disabled="isTriggering">
        {{ isTriggering ? '触发中...' : '🔄 手动触发抓取' }}
      </button>
      <button @click="fetchReports">🔍 刷新报告</button>
    </div>

    <div v-if="loading" class="loading">加载中...</div>

    <div v-else class="report-list">
      <div v-for="report in reports" :key="report.id" class="report-card">
        <div class="report-header">
          <span class="report-title">{{ report.title }}</span>
          <span class="report-time">{{ formatTime(report.created_at) }}</span>
        </div>
        <p class="report-summary">{{ report.summary }}</p>
        <div v-if="report.keywords" class="report-keywords">
          <span v-for="kw in report.keywords.split(',')" :key="kw" class="keyword-tag">
            {{ kw.trim() }}
          </span>
        </div>
        <a :href="report.source_url" target="_blank" class="report-link">📎 查看原文</a>
      </div>

      <div v-if="reports.length === 0" class="empty">
        暂无报告，请等待定时任务执行或手动触发抓取。
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const API_BASE = 'https://monitor.chusxxin.cyou';  // 换你的自定义域名

const reports = ref([]);
const loading = ref(true);
const isTriggering = ref(false);

const fetchReports = async () => {
  loading.value = true;
  try {
    const res = await fetch(`${API_BASE}/reports`);
    reports.value = await res.json();
  } catch (err) {
    console.error('获取报告失败:', err);
  } finally {
    loading.value = false;
  }
};

const triggerFetch = async () => {
  isTriggering.value = true;
  try {
    await fetch(`${API_BASE}/trigger`, { method: 'POST' });
    // 3秒后自动刷新
    setTimeout(() => fetchReports(), 3000);
  } catch (err) {
    console.error('触发失败:', err);
  } finally {
    isTriggering.value = false;
  }
};

const formatTime = (timestamp) => {
  if (!timestamp) return '';
  const date = new Date(timestamp);
  return date.toLocaleString('zh-CN');
};

onMounted(fetchReports);
</script>

<style scoped>
.monitor-container {
  max-width: 860px;
  margin: 0 auto;
  padding: 40px 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: #fafafa;
  min-height: 100vh;
}

h1 { font-size: 2rem; font-weight: 600; color: #171717; margin-bottom: 4px; }
.subtitle { color: #6b7280; font-size: 0.95rem; margin-bottom: 28px; }

.controls {
  display: flex; gap: 12px; margin-bottom: 24px;
}

button {
  padding: 10px 20px;
  border: 1px solid #e5e7eb;
  border-radius: 30px;
  background: #fff;
  font-size: 0.9rem;
  cursor: pointer;
  color: #171717;
  transition: all 0.15s;
}
button:hover:not(:disabled) {
  background: #f3f4f6; border-color: #171717;
}
button:disabled { opacity: 0.5; cursor: not-allowed; }

.loading { text-align: center; color: #9ca3af; padding: 60px 0; }

.report-list { display: flex; flex-direction: column; gap: 16px; }

.report-card {
  background: #fff;
  border-radius: 16px;
  padding: 20px 24px;
  border: 1px solid #f0f0f2;
  box-shadow: 0 2px 8px rgba(0,0,0,0.02);
}
.report-header {
  display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px;
}
.report-title { font-weight: 600; color: #171717; font-size: 1.05rem; }
.report-time { font-size: 0.8rem; color: #9ca3af; }

.report-summary { color: #374151; line-height: 1.6; margin-bottom: 12px; }

.report-keywords { display: flex; gap: 8px; flex-wrap: wrap; margin-bottom: 12px; }
.keyword-tag {
  background: #f3f4f6;
  color: #4b5563;
  padding: 4px 12px;
  border-radius: 30px;
  font-size: 0.8rem;
}
.report-link { color: #18181b; font-size: 0.85rem; text-underline-offset: 3px; }

.empty { text-align: center; color: #9ca3af; padding: 60px 0; }
</style>