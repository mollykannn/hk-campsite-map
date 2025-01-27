<template>
  <div class="side-panel" :class="{ 'is-open': isOpen }">
    <button class="close-button" @click="$emit('close')" v-if="isOpen">
      <span class="close-icon">×</span>
    </button>
    <div v-if="content" class="panel-content" v-html="content"></div>
  </div>
</template>

<script>
export default {
  name: 'SidePanel',
  props: {
    isOpen: {
      type: Boolean,
      default: false
    },
    content: {
      type: String,
      default: ''
    }
  },
  emits: ['close']
}
</script>

<style scoped>
.side-panel {
  position: fixed;
  left: -100%;
  top: 0;
  width: min(400px, 90vw); /* 最大寬度400px，但不超過螢幕寬度的90% */
  height: 100vh;
  background: white;
  box-shadow: 2px 0 5px rgba(0,0,0,0.1);
  transition: left 0.3s ease;
  z-index: 1000;
  overflow-y: auto;
}

.side-panel.is-open {
  left: 0;
}

.panel-content {
  padding: 20px;
  font-size: max(14px, min(16px, 3.5vw)); /* 自適應字體大小 */
}

.close-button {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 30px;
  height: 30px;
  border: none;
  background: transparent;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  z-index: 1001;
}

.close-icon {
  font-size: 24px;
  color: #666;
  line-height: 1;
}

.close-button:hover .close-icon {
  color: #333;
}

/* 手機版樣式調整 */
@media (max-width: 480px) {
  .panel-content {
    padding: 15px;
  }

  .close-button {
    top: 5px;
    right: 5px;
  }

  :deep(.info-window h3) {
    font-size: 1.2em;
    margin-bottom: 10px;
  }

  :deep(.info-window p) {
    margin: 6px 0;
  }
}
</style> 