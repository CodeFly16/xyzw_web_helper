<template>
  <div class="default-layout">
    <!-- 顶部导航 -->
    <nav class="dashboard-nav">
      <div class="nav-container">
        <div class="nav-brand">
          <img src="/icons/xiaoyugan.png" alt="XYZW" class="brand-logo" />
          <div class="brand-toggle" @click="isMobileMenuOpen = true">
            <n-icon>
              <Menu />
            </n-icon>
            <span class="brand-text">XYZW 控制台</span>
          </div>
        </div>

        <div class="nav-menu">
          <router-link
            to="/admin/dashboard"
            class="nav-item"
            active-class="active"
          >
            <n-icon>
              <Home />
            </n-icon>
            <span>首页</span>
          </router-link>
          <router-link
            to="/admin/game-features"
            class="nav-item"
            active-class="active"
          >
            <n-icon>
              <Cube />
            </n-icon>
            <span>游戏功能</span>
          </router-link>
          <router-link to="/tokens" class="nav-item" active-class="active">
            <n-icon>
              <PersonCircle />
            </n-icon>
            <span>Token管理</span>
          </router-link>
          <router-link
            to="/admin/batch-daily-tasks"
            class="nav-item"
            active-class="active"
          >
            <n-icon>
              <Layers />
            </n-icon>
            <span>批量日常</span>
          </router-link>
          <router-link
            to="/admin/message-test"
            class="nav-item"
            active-class="active"
          >
            <n-icon>
              <ChatbubbleEllipsesSharp />
            </n-icon>
            <span>消息测试</span>
          </router-link>
          <router-link to="/admin/legion-war" class="nav-item" active-class="active"  v-if="isNowInLegionWarTime()" >
            <n-icon>
              <LockOpen />
            </n-icon>
            <span>实时盐场</span>
          </router-link>
        </div>

        <div class="nav-user">
          <!-- 主题切换按钮 -->
          <ThemeToggle />

          <n-dropdown :options="userMenuOptions" @select="handleUserAction">
            <div class="user-info">
              <n-avatar
                :src="selectedToken?.avatar || '/icons/xiaoyugan.png'"
                size="medium"
                fallback-src="/icons/xiaoyugan.png"
              />
              <span class="username">{{
                selectedToken?.name || "未选择Token"
              }}</span>
              <n-icon>
                <ChevronDown />
              </n-icon>
            </div>
          </n-dropdown>
        </div>
      </div>
    </nav>
    <n-drawer
      v-model:show="isMobileMenuOpen"
      placement="left"
      style="width: 260px"
    >
      <div class="drawer-menu">
        <router-link
          to="/admin/dashboard"
          class="drawer-item"
          @click="isMobileMenuOpen = false"
        >
          <n-icon>
            <Home />
          </n-icon>
          <span>首页</span>
        </router-link>
        <router-link
          to="/admin/game-features"
          class="drawer-item"
          @click="isMobileMenuOpen = false"
        >
          <n-icon>
            <Cube />
          </n-icon>
          <span>游戏功能</span>
        </router-link>
        <router-link
          to="/tokens"
          class="drawer-item"
          @click="isMobileMenuOpen = false"
        >
          <n-icon>
            <PersonCircle />
          </n-icon>
          <span>Token管理</span>
        </router-link>
        <router-link
          to="/admin/daily-tasks"
          class="drawer-item"
          @click="isMobileMenuOpen = false"
        >
          <n-icon>
            <Settings />
          </n-icon>
          <span>任务管理</span>
        </router-link>
        <router-link
          to="/admin/batch-daily-tasks"
          class="drawer-item"
          @click="isMobileMenuOpen = false"
        >
          <n-icon>
            <Layers />
          </n-icon>
          <span>批量日常</span>
        </router-link>
        <router-link
          to="/admin/message-test"
          class="drawer-item"
          @click="isMobileMenuOpen = false"
        >
          <n-icon>
            <ChatbubbleEllipsesSharp />
          </n-icon>
          <span>消息测试</span>
        </router-link>
          <router-link to="/admin/legion-war" class="nav-item" active-class="active"  v-if="isNowInLegionWarTime()" >
            <n-icon>
              <LockOpen />
            </n-icon>
            <span>实时盐场</span>
          </router-link>
        <router-link
          to="/admin/profile"
          class="drawer-item"
          @click="isMobileMenuOpen = false"
        >
          <n-icon>
            <Settings />
          </n-icon>
          <span>个人设置</span>
        </router-link>
      </div>
    </n-drawer>
    <div class="main">
      <router-view />
    </div>

    <!-- 断线蒙层 -->
    <Transition name="overlay-fade">
      <div v-if="isDisconnected" class="disconnect-overlay">
        <div class="disconnect-card">
          <div class="disconnect-icon">
            <n-icon size="56" color="#f5a623">
              <WifiOutline />
            </n-icon>
          </div>
          <h2 class="disconnect-title">连接已断开</h2>
          <p class="disconnect-desc">
            与服务器的 WebSocket 连接已断开，请点击下方按钮重新连接。
          </p>
          <n-button
            type="primary"
            size="large"
            :loading="isReconnecting"
            @click="handleReconnect"
            class="reconnect-btn"
          >
            {{ isReconnecting ? '连接中...' : '重新连接' }}
          </n-button>
        </div>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import {
  useTokenStore,
  selectedToken,
  selectedTokenId,
} from "@/stores/tokenStore";
import ThemeToggle from "@/components/Common/ThemeToggle.vue";
import {
  Home,
  PersonCircle,
  Cube,
  Settings,
  ChevronDown,
  ChatbubbleEllipsesSharp,
  LockClosedSharp,LockOpen,
  Menu,
  Layers,
  WifiOutline,
} from "@vicons/ionicons5";

import { useRouter } from 'vue-router'
import { useMessage } from 'naive-ui'
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'
import { isNowInLegionWarTime } from '@/utils/clubBattleUtils'

const tokenStore = useTokenStore();
const router = useRouter();
const message = useMessage();

const isMobileMenuOpen = ref(false);

// ---- 断线检测 ----
const isReconnecting = ref(false);

// 当前选中 token 的 ws 状态
const currentWsStatus = computed(() => {
  if (!selectedTokenId.value) return null;
  return tokenStore.wsConnections[selectedTokenId.value]?.status ?? null;
});

// 仅在有 token 且状态为 disconnected/error 时显示蒙层
const isDisconnected = computed(() => {
  if (!selectedTokenId.value) return false;
  const s = currentWsStatus.value;
  return s === 'disconnected' || s === 'error';
});

// 心跳定时器：每 15s 检测一次，若断开则强制刷新状态（ws 内部会自动重连，这里只做 UI 兜底）
let heartbeatTimer = null;
const startHeartbeat = () => {
  stopHeartbeat();
  heartbeatTimer = setInterval(() => {
    const tokenId = selectedTokenId.value;
    if (!tokenId) return;
    const status = tokenStore.getWebSocketStatus(tokenId);
    // 若已断线，触发重连
    if (status === 'disconnected' || status === 'error') {
      // 状态已是响应式，computed 自动更新，蒙层会显示
    }
  }, 15000);
};
const stopHeartbeat = () => {
  if (heartbeatTimer) {
    clearInterval(heartbeatTimer);
    heartbeatTimer = null;
  }
};

// 重新连接
const handleReconnect = () => {
  const tokenId = selectedTokenId.value;
  if (!tokenId || isReconnecting.value) return;
  isReconnecting.value = true;

  // selectToken(id, forceReconnect=true) 会取出 token 数据并创建连接
  tokenStore.selectToken(tokenId, true);

  // 监听状态变化，连接成功/失败后还原 loading
  const stop = watch(currentWsStatus, (status) => {
    if (status === 'connected') {
      message.success('重新连接成功');
      isReconnecting.value = false;
      stop();
    } else if (status === 'error') {
      message.error('连接失败，请稍后重试');
      isReconnecting.value = false;
      stop();
    }
  });

  // 15s 超时保护，防止一直 loading
  setTimeout(() => {
    if (isReconnecting.value) {
      isReconnecting.value = false;
      stop();
    }
  }, 15000);
};

// 切换 token 时重置状态
watch(selectedTokenId, () => {
  isReconnecting.value = false;
});

onMounted(() => startHeartbeat());
onUnmounted(() => stopHeartbeat());

const userMenuOptions = [
  {
    label: "清除所有Token并退出",
    key: "logout",
  },
];

// 方法
const handleUserAction = async (key) => {
  switch (key) {
    case "logout":
      await tokenStore.clearAllTokens();
      message.success("已清除所有Token");
      router.push("/tokens");
      break;
  }
};
</script>

<style scoped lang="scss">
// 导航栏
.dashboard-nav {
  background: var(--bg-primary);
  border-bottom: 1px solid var(--border-light);
  padding: 0 var(--spacing-lg);
  position: sticky;
  top: 0;
  z-index: var(--z-sticky);
}

.nav-container {
  display: flex;
  align-items: center;
  height: 64px;
  max-width: 1400px;
  margin: 0 auto;
}

.nav-brand {
  display: flex;
  align-items: center;
  gap: var(--spacing-xs);
  margin-right: var(--spacing-xl);
}

.brand-logo {
  width: 40px;
  height: 40px;
  border-radius: var(--border-radius-small);
}

.brand-text {
  font-size: var(--font-size-lg);
  font-weight: var(--font-weight-semibold);
  color: var(--text-primary);
}

.brand-toggle {
  display: none;
  align-items: center;
  gap: var(--spacing-xs);
  cursor: pointer;
  font-size: var(--font-size-lg);
}

.brand-toggle .n-icon {
  font-size: inherit;
}

.nav-menu {
  display: flex;
  gap: var(--spacing-md);
  flex: 1;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: var(--spacing-xs);
  padding: var(--spacing-sm) var(--spacing-md);
  border-radius: var(--border-radius-medium);
  color: var(--text-secondary);
  text-decoration: none;
  transition: all var(--transition-fast);

  &:hover {
    background: var(--bg-tertiary);
    color: var(--text-primary);
  }

  &.active {
    background: var(--primary-color-light);
    color: var(--primary-color);
  }
}

.nav-user {
  margin-left: auto;
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
}

.user-info {
  display: flex;
  align-items: center;
  gap: var(--spacing-sm);
  padding: var(--spacing-sm);
  border-radius: var(--border-radius-medium);
  cursor: pointer;
  transition: background var(--transition-fast);

  &:hover {
    background: var(--bg-tertiary);
  }
}

.username {
  font-weight: var(--font-weight-medium);
  color: var(--text-primary);
}

@media (max-width: 768px) {
  .nav-item span {
    display: none;
  }

  .nav-menu {
    display: none;
  }

  .nav-item {
    padding: var(--spacing-sm);
    flex: 0 0 auto;
  }

  .username {
    display: none;
  }

  .nav-container {
    height: 56px;
  }

  .brand-logo {
    display: none;
  }

  .brand-toggle {
    display: inline-flex;
  }
}

.drawer-menu {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-sm);
  padding: var(--spacing-md);
}

.drawer-item {
  display: flex;
  align-items: center;
  gap: var(--spacing-sm);
  padding: var(--spacing-sm) var(--spacing-md);
  border-radius: var(--border-radius-medium);
  color: var(--text-secondary);
  text-decoration: none;
}

.drawer-item.router-link-active {
  background: var(--primary-color-light);
  color: var(--primary-color);
}

// ---- 断线蒙层 ----
.disconnect-overlay {
  position: fixed;
  inset: 0;
  z-index: 9999;
  background: rgba(0, 0, 0, 0.65);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
}

.disconnect-card {
  background: var(--bg-primary);
  border-radius: 16px;
  padding: 48px 40px;
  text-align: center;
  max-width: 380px;
  width: 90%;
  box-shadow: 0 24px 64px rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

.disconnect-icon {
  animation: pulse 2s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.6; transform: scale(0.92); }
}

.disconnect-title {
  margin: 0;
  font-size: 22px;
  font-weight: 600;
  color: var(--text-primary);
}

.disconnect-desc {
  margin: 0;
  font-size: 14px;
  color: var(--text-secondary);
  line-height: 1.6;
}

.reconnect-btn {
  margin-top: 8px;
  min-width: 140px;
}

// ---- 蒙层动画 ----
.overlay-fade-enter-active,
.overlay-fade-leave-active {
  transition: opacity 0.3s ease;
}
.overlay-fade-enter-from,
.overlay-fade-leave-to {
  opacity: 0;
}

/* 禁用样式：灰化、鼠标禁止、无hover效果 */
.nav-item.disabled {
  background: #cccccc;
  color: #999999;
  cursor: not-allowed; /* 鼠标样式：禁止 */
  pointer-events: none; /* 可选：直接禁用所有鼠标事件（比阻止click更彻底） */
}
</style>
