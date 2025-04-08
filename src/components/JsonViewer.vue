<template>
  <div class="json-viewer-container">
    <button class="copy-btn" @click="copyToClipboard" title="复制到剪贴板">
      <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect>
        <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path>
      </svg>
    </button>
    <pre class="json-output">{{ formattedJson }}</pre>
  </div>
</template>

<script>
export default {
  name: 'JsonViewer',
  props: {
    data: {
      type: [Object, Array, String],
      required: true
    }
  },
  computed: {
    formattedJson() {
      try {
        const data = typeof this.data === 'string' ? JSON.parse(this.data) : this.data;
        return JSON.stringify(data, null, 2);
      } catch (e) {
        return `Error: ${e.message}`;
      }
    }
  },
  methods: {
    copyToClipboard() {
      navigator.clipboard.writeText(this.formattedJson)
        .then(() => {
          // 可以添加复制成功提示
          alert('已复制到剪贴板');
        })
        .catch(err => {
          console.error('复制失败:', err);
          alert('复制失败，请手动选择并复制');
        });
    }
  }
}
</script>

<style scoped>
.json-viewer-container {
  position: relative;
  width: 100%;
  height: 100%;
}

.copy-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  background: rgba(255, 255, 255, 0.2);
  border: none;
  border-radius: 4px;
  color: #f8f8f2;
  padding: 5px;
  cursor: pointer;
  z-index: 10;
  transition: all 0.2s ease;
}

.copy-btn:hover {
  background: rgba(255, 255, 255, 0.4);
}

.json-output {
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', 'Consolas', monospace;
  font-size: 14px;
  line-height: 1.5;
  margin: 0;
  padding: 10px;
  color: #f8f8f2;
  white-space: pre-wrap;
  text-align: left;
  height: 100%;
  overflow: auto;
}
</style>