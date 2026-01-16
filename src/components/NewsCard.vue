<template>
  <div
    class="news-card glass-effect"
    :class="{ expanded: isExpanded }"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
    @click="handleCapsuleClick"
  >
    <!-- 胶囊模式内容 -->
    <div class="capsule-content">
      <span class="icon">🔥</span>
      <div class="carousel" v-if="newsList.length > 0">
        <Transition name="slide-up" mode="out-in">
          <span :key="carouselIndex" class="carousel-text">
            {{ newsList[carouselIndex]?.title }}
          </span>
        </Transition>
      </div>
      <span v-else class="carousel-text">热榜加载中...</span>
    </div>

    <!-- 完整模式内容 -->
    <div class="expanded-content" @click.stop>
      <!-- 关闭按钮 -->
      <div class="close-btn" @click.stop="handleClose" title="收起">
        <svg viewBox="0 0 24 24" width="18" height="18" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg>
      </div>

      <div class="card-header">
        <h3>🔥 今日热榜</h3>
        <div class="tabs">
          <span
            v-for="type in sources"
            :key="type.value"
            :class="{ active: currentType === type.value }"
            @click.stop="changeSource(type.value)"
            :title="type.name"
          >
            {{ type.shortName }}
          </span>
        </div>
      </div>

      <div class="content-area">
        <div v-if="loading" class="state-box loading">
          <div class="spinner"></div>
          <span>正在获取热点...</span>
        </div>

        <div v-else-if="isError" class="state-box error">
          <span>加载失败</span>
          <button class="retry-btn" @click="fetchNews(currentType)">重试</button>
        </div>

        <div v-else class="news-list">
          <a
            v-for="(item, index) in newsList"
            :key="index"
            :href="item.link"
            target="_blank"
            class="news-item"
            @click.stop
          >
            <span class="index" :class="'top-' + (index + 1)">{{ index + 1 }}</span>
            <span class="title" :title="item.title">{{ item.title }}</span>
            <span class="hot-val" v-if="item.hotValue">{{ formatHotValue(item.hotValue) }}</span>
          </a>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';

const loading = ref(true);
const isError = ref(false);
const newsList = ref([]);
const currentType = ref('wb');

// 缓存 Map: { [type]: { data: [], timestamp: number } }
const newsCache = new Map();
const CACHE_DURATION = 5 * 60 * 1000; // 5分钟

// 交互状态
const isExpanded = ref(false);
const carouselIndex = ref(0);
let hoverTimer = null;
let carouselTimer = null;

const sources = [
  { name: '微博', value: 'wb', shortName: 'WB' },
  { name: '知乎', value: 'zhihu', shortName: 'ZH' },
  { name: '少数派', value: 'sspai', shortName: 'SS' },
  { name: 'IT之家', value: 'it', shortName: 'IT' },
  { name: '虎扑', value: 'hupu', shortName: 'HP' },
  { name: 'V2EX', value: 'v2ex', shortName: 'V2' },
  { name: '36氪', value: '36kr', shortName: '36K' },
  { name: '哔哩哔哩', value: 'bilibili', shortName: 'Bili' }
];

const typeMap = {
  wb: 'weibo',
  zhihu: 'zhihu',
  sspai: 'sspai',
  it: 'ithome',
  hupu: 'hupu',
  v2ex: 'v2ex',
  '36kr': '36kr',
  bilibili: 'bilibili'
};

// 交互逻辑
const handleMouseEnter = () => {
  clearTimeout(hoverTimer);
  hoverTimer = setTimeout(() => {
    isExpanded.value = true;
  }, 300); // 300ms 防抖
};

const handleMouseLeave = () => {
  clearTimeout(hoverTimer);
  isExpanded.value = false;
};

// 专门处理点击事件，适配移动端点击展开
const handleCapsuleClick = () => {
  if (!isExpanded.value) {
    isExpanded.value = true;
  }
};

const handleClose = () => {
  clearTimeout(hoverTimer);
  isExpanded.value = false;
};

// 轮播逻辑
const startCarousel = () => {
  stopCarousel();
  carouselTimer = setInterval(() => {
    if (newsList.value.length > 0) {
      // 轮播前3条
      carouselIndex.value = (carouselIndex.value + 1) % Math.min(newsList.value.length, 3);
    }
  }, 5000);
};

const stopCarousel = () => {
  if (carouselTimer) clearInterval(carouselTimer);
};

const changeSource = (type) => {
  if (currentType.value === type) return;
  currentType.value = type;
  fetchNews(type);
};

const formatHotValue = (val) => {
  if (!val) return '';
  return val;
};

const fetchNews = async (type) => {
  loading.value = true;
  isError.value = false;
  
  // 检查缓存
  const cached = newsCache.get(type);
  if (cached && Date.now() - cached.timestamp < CACHE_DURATION) {
    console.log(`[Cache Hit] ${type}`);
    newsList.value = cached.data;
    loading.value = false;
    // 重置轮播
    carouselIndex.value = 0;
    return;
  }
  
  const apiType = typeMap[type] || 'weibo';

  try {
    const res = await fetch(`/api/?platform=${apiType}`);
    const data = await res.json();
    
    const list = data.data || [];
    
    if (list && list.length > 0) {
      const formattedList = list.slice(0, 10).map(item => ({
        title: item.title,
        link: item.url,
        hotValue: item.hot || item.score || item.views || ''
      }));
      
      newsList.value = formattedList;
      
      // 写入缓存
      newsCache.set(type, {
        data: formattedList,
        timestamp: Date.now()
      });
      
      // 重置轮播
      carouselIndex.value = 0;
    } else {
      if (Array.isArray(data)) {
         const formattedList = data.slice(0, 10).map(item => ({
          title: item.title,
          link: item.url,
          hotValue: item.hot || item.score || item.views || ''
        }));
        newsList.value = formattedList;
        newsCache.set(type, { data: formattedList, timestamp: Date.now() });
      } else {
         throw new Error('Data format error');
      }
    }
  } catch (error) {
    console.error('新闻加载失败', error);
    isError.value = true;
  } finally {
    loading.value = false;
  }
};

onMounted(() => {
  fetchNews('wb');
  startCarousel();
});

onBeforeUnmount(() => {
  stopCarousel();
  clearTimeout(hoverTimer);
});
</script>

<style scoped>
.news-card {
  position: fixed;
  /* 初始胶囊尺寸 */
  width: 260px;
  height: 44px;
  padding: 0;
  border-radius: 22px;
  
  color: #fff;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 4px 16px 0 rgba(0, 0, 0, 0.1);
  
  /* 弹性动画 */
  transition: all 0.4s cubic-bezier(0.25, 0.8, 0.25, 1);
  z-index: 100;
  overflow: hidden;
  box-sizing: border-box;
}

/* 展开状态尺寸 */
.news-card.expanded {
  width: 360px !important;
  height: 500px !important;
  border-radius: 20px;
}

/* 移动端适配 */
/* 移动端适配 */
@media (max-width: 768px) {
  .news-card {
    /* 移动端修复：强制宽度与布局防止坍塌 */
    width: 85vw;
    max-width: 400px;
    min-width: 280px;
    height: 44px;
    
    display: flex;
    flex-direction: row;
    align-items: center;

    background: rgba(20, 20, 20, 0.65);
    backdrop-filter: blur(12px);
    -webkit-backdrop-filter: blur(12px);
    
    border: 1px solid rgba(255, 255, 255, 0.15);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
    border-radius: 20px;
    
    padding: 0;
    transform-origin: bottom center;
  }

  /* 修复文字溢出导致宽度的计算问题 */
  .news-card .carousel {
    flex: 1;
    width: 0;
  }
  
  .news-card .capsule-content {
     padding: 0 12px; /* 移动端内边距 */
     justify-content: flex-start; /* 左对齐 */
  }

  .news-card .carousel-text {
    color: #fff; /* 纯白文字 */
    font-size: 13px; /* 字号稍小 */
    text-align: left;
    margin-left: 8px; /* 这里的 margin 配合 justify-content: flex-start */
  }
  
  .news-card.expanded {
    width: 90vw !important;
    height: 60vh !important;
    background: rgba(40, 40, 40, 0.85);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 24px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  }
}

/* 胶囊内容 */
.capsule-content {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0 16px;
  opacity: 1;
  transition: opacity 0.3s ease;
  pointer-events: none; /* 让点击穿透给父级 */
}

.news-card.expanded .capsule-content {
  opacity: 0;
}

/* 展开后的内容 */
.expanded-content {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  padding: 20px;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.3s ease;
  box-sizing: border-box;
}

.news-card.expanded .expanded-content {
  opacity: 1;
  pointer-events: auto;
}

/* 关闭按钮 */
.close-btn {
  position: absolute;
  top: 15px;
  right: 15px;
  cursor: pointer;
  color: rgba(255, 255, 255, 0.6);
  z-index: 10;
  padding: 4px;
  border-radius: 50%;
  border: 1px solid transparent; 
  transition: all 0.2s;
  display: flex;
}
.close-btn:hover {
  background: rgba(255,255,255,0.2);
  color: #fff;
}

.icon {
  margin-right: 8px;
}

.carousel {
  flex: 1;
  position: relative;
  height: 20px;
  overflow: hidden;
}

.carousel-text {
  display: inline-block;
  font-size: 14px;
  width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* 头部样式 */
.card-header {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  margin-bottom: 5px;
  padding-bottom: 10px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  height: 40px;
}

h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 6px;
  margin-right: auto;
}

/* Tabs 样式 */
.tabs {
  display: flex;
  gap: 10px; /* 增加间距 */
  background: rgba(0, 0, 0, 0.2);
  padding: 4px;
  border-radius: 8px;
  margin-right: 24px;
  overflow-x: auto; /* 允许横向滚动 */
  scrollbar-width: none; /* Firefox 隐藏滚动条 */
  -ms-overflow-style: none; /* IE 10+ */
  white-space: nowrap;
  /* 增加渐变遮罩提示 */
  mask-image: linear-gradient(to right, black 90%, transparent 100%);
  -webkit-mask-image: linear-gradient(to right, black 90%, transparent 100%);
}

.tabs::-webkit-scrollbar {
  display: none; /* Chrome Safari 隐藏滚动条 */
}

.tabs span {
  font-size: 12px;
  padding: 4px 8px;
  cursor: pointer;
  border-radius: 6px;
  color: rgba(255, 255, 255, 0.7);
  transition: all 0.2s;
  min-width: 24px;
  text-align: center;
  flex-shrink: 0; /* 防止挤压 */
}

.tabs span:hover {
  color: #fff;
}

.tabs span.active {
  color: #333;
  background: rgba(255, 255, 255, 0.9);
  font-weight: 600;
}

/* 内容区域 */
.content-area {
  flex: 1;
  overflow-y: auto;
  scrollbar-width: none; 
  -ms-overflow-style: none;
  margin-top: 10px;
}
.content-area::-webkit-scrollbar {
  display: none;
}

/* 列表样式 */
.news-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.news-item {
  display: flex;
  align-items: center;
  text-decoration: none;
  color: rgba(255, 255, 255, 0.9);
  padding: 8px;
  border-radius: 8px;
  transition: all 0.2s;
  font-size: 14px;
}

.news-item:hover {
  background: rgba(255, 255, 255, 0.15);
  transform: translateX(2px);
  color: #fff;
}

.index {
  width: 20px;
  font-weight: 700;
  font-family: 'JetBrains Mono', monospace;
  opacity: 0.6;
  text-align: center;
  margin-right: 10px;
  font-size: 14px;
}

.top-1 { color: #ff4757; opacity: 1; font-size: 16px; }
.top-2 { color: #ffa502; opacity: 1; font-size: 16px; }
.top-3 { color: #eccc68; opacity: 1; font-size: 16px; }

.title {
  flex: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin-right: 8px;
}

.hot-val {
  font-size: 12px;
  opacity: 0.5;
  font-family: 'JetBrains Mono', monospace;
  white-space: nowrap;
}

/* 状态样式 */
.state-box {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px 0;
  color: rgba(255, 255, 255, 0.7);
  font-size: 14px;
  gap: 12px;
}

.spinner {
  width: 24px;
  height: 24px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.retry-btn {
  background: rgba(255, 255, 255, 0.2);
  border: none;
  color: #fff;
  padding: 6px 16px;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.2s;
  font-size: 13px;
}

.retry-btn:hover {
  background: rgba(255, 255, 255, 0.3);
}

/* 动画 */
.slide-up-enter-active,
.slide-up-leave-active {
  transition: all 0.3s ease;
  position: absolute; /* 确保位置重叠 */
  width: 100%;
}

.slide-up-enter-from {
  opacity: 0;
  transform: translateY(100%);
}

.slide-up-leave-to {
  opacity: 0;
  transform: translateY(-100%);
}
</style>