<template>
  <div :class="status.siteStatus !== 'normal' ? 'cover focus' : 'cover'">
    <img
      v-show="status.imgLoadStatus"
      :key="bgUrl"
      class="background"
      alt="background"
      :src="bgUrl"
      :style="{ '--blur': set.backgroundBlur + 'px' }"
      @load="imgLoadComplete"
      @error.once="imgLoadError"
      @animationend="imgAnimationEnd"
    />
    <Transition name="fade">
      <div v-if="set.showBackgroundGray" class="gray" />
    </Transition>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import { statusStore, setStore } from "@/stores";

const set = setStore();
const status = statusStore();
const bgUrl = ref(null);
const imgTimeout = ref(null);
const emit = defineEmits(["loadComplete"]);

// 壁纸随机数
// 请依据文件夹内的图片个数修改 Math.random() 后面的第一个数字
const bgRandom = Math.floor(Math.random() * 3 + 1);

// 赋值壁纸
const setBgUrl = (isRefresh = false) => {
  const { backgroundType } = set;
  let url = "";

  switch (backgroundType) {
    case 0:
      url = `/background/bg${bgRandom}.jpg`;
      break;
    case 1: {
      const isMobile = window.innerWidth < 768;
      url = `https://www.yumus.cn/api/?brand=bing&ua=${
        isMobile ? "m" : "pc"
      }`;
      break;
    }
    case 2:
      // 风景大片
      url = "https://www.yumus.cn/api/?target=img&brand=360&type=3";
      break;
    case 3:
      // 动漫卡通
      url = "https://www.yumus.cn/api/?target=img&brand=360&type=5";
      break;
    case 4:
      // 4K专区
      url = "https://www.yumus.cn/api/?target=img&brand=360&type=0";
      break;
    case 5:
      // 美女模特
      url = "https://www.yumus.cn/api/?target=img&brand=360&type=1";
      break;
    case 6:
      // 爱情美图
      url = "https://www.yumus.cn/api/?target=img&brand=360&type=2";
      break;
    case 7:
      // 小清新
      url = "https://www.yumus.cn/api/?target=img&brand=360&type=4";
      break;
    case 99:
      url = set.backgroundCustom;
      break;
    default:
      url = `/background/bg${bgRandom}.jpg`;
      break;
  }

  // 如果是 API 地址，添加时间戳参数以防止缓存
  if (isRefresh && url.includes("http")) {
    const separator = url.includes("?") ? "&" : "?";
    url = `${url}${separator}t=${new Date().getTime()}`;
  }

  bgUrl.value = url;
};

// 图片加载完成
const imgLoadComplete = () => {
  imgTimeout.value = setTimeout(
    () => {
      status.setImgLoadStatus(true);
    },
    Math.floor(Math.random() * (600 - 300 + 1)) + 300,
  );
};

// 图片动画完成
const imgAnimationEnd = () => {
  console.log("壁纸加载且动画完成");
  // 加载完成事件
  emit("loadComplete");
};

// 图片显示失败
const imgLoadError = () => {
  console.error("壁纸加载失败：", bgUrl.value);
  $message.error("壁纸加载失败，已临时切换回默认");
  bgUrl.value = `/background/bg${bgRandom}.jpg`;
};

onMounted(() => {
  setBgUrl();
});

onBeforeUnmount(() => {
  clearTimeout(imgTimeout.value);
});

defineExpose({
  setBgUrl,
});
</script>

<style lang="scss" scoped>
.cover {
  width: 100%;
  height: 100%;
  position: relative;
  background-color: var(--body-background-color);
  &.focus {
    .background {
      filter: blur(calc(var(--blur) + 10px)) brightness(0.8);
      transform: scale(1.3);
    }
  }
  .background {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    backface-visibility: hidden;
    transform: scale(1.2);
    filter: blur(var(--blur));
    transition:
      filter 0.3s,
      transform 0.3s;
    animation: fade-blur-in 1s cubic-bezier(0.25, 0.46, 0.45, 0.94);
  }
  .gray {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    background-image: radial-gradient(rgba(0, 0, 0, 0) 0, rgba(0, 0, 0, 0.5) 100%),
      radial-gradient(rgba(0, 0, 0, 0) 33%, rgba(0, 0, 0, 0.3) 166%);
  }
}
</style>
