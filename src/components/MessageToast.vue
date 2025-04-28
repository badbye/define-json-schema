<template>
  <transition name="toast-fade">
    <div v-if="visible" class="message-toast" :class="type">
      {{ localMessage }}
    </div>
  </transition>
</template>

<script>
export default {
  name: 'MessageToast',
  props: {
    message: {
      type: String,
      default: ''
    },
    duration: {
      type: Number,
      default: 2000
    },
    type: {
      type: String,
      default: 'info', // 可选值：info, success, error, warning
      validator: (value) => ['info', 'success', 'error', 'warning'].includes(value)
    }
  },
  data() {
    return {
      visible: false,
      timer: null,
      localMessage: this.message
    }
  },
  watch: {
    // 监听 message prop 的变化，并更新本地变量
    message(newMessage) {
      this.localMessage = newMessage;
    }
  },
  methods: {
    show() {
      clearTimeout(this.timer);
      this.visible = true;
      
      this.timer = setTimeout(() => {
        this.hide();
      }, this.duration);
    },
    hide() {
      this.visible = false;
    }
  },
  beforeUnmount() {
    clearTimeout(this.timer);
  }
}
</script>

<style scoped>
.message-toast {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  padding: 10px 20px;
  border-radius: 4px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  font-size: 14px;
  z-index: 9999;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}

.message-toast.success {
  background: rgba(40, 167, 69, 0.9);
}

.message-toast.error {
  background: rgba(220, 53, 69, 0.9);
}

.message-toast.warning {
  background: rgba(255, 193, 7, 0.9);
}

.toast-fade-enter-active,
.toast-fade-leave-active {
  transition: all 0.3s ease;
}

.toast-fade-enter-from,
.toast-fade-leave-to {
  opacity: 0;
  transform: translateX(-50%) translateY(-20px);
}
</style>