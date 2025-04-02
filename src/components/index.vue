<template>
  <n-global-style />
  <n-card class="notice" size="small" :style="theme()">
    <n-switch
      :default-value="isDarkTheme"
      @update:value="(v) => $emit('changeIsDarkTheme', v)"
      size="small"
    >
      <template #checked-icon>
        <n-icon :component="Sparkles" />
      </template>
      <template #unchecked-icon>
        <n-icon :component="SunnySharp" />
      </template>
    </n-switch>
    <div v-html="notice"></div>
  </n-card>

  <!-- 使用原来的网格布局 -->
  <n-grid
    x-gap="10"
    y-gap="10"
    cols="2 s:3 m:4 l:5 xl:5 2xl:6"
    responsive="screen"
  >
    <n-grid-item
      class="cardclss no-select"
      v-for="item in itemslist"
      :key="item.carID"
    >
      <!-- 完全自定义的卡片大小骨架屏 -->
      <div v-if="item.loading" class="card-skeleton">
        <div class="skeleton-header">
          <div class="skeleton-badge"></div>
          <div class="skeleton-title"></div>
        </div>
        <div class="skeleton-status"></div>
        <div class="skeleton-progress"></div>
      </div>

      <!-- 真实内容 -->
      <n-card v-else size="small" @click="redirectTo(item.carID)">
        <div class="box">
          <div class="type plus-badge" :style="{ background: item.labelColor }">
            {{ item.label != "" ? item.label : "NO" }}
          </div>

          <div class="cartitle">
            <n-text class="title">{{ item.carID }}</n-text>
          </div>
        </div>

        <div class="message-with-dot">实时状态：{{ item.message }}</div>

        <div :style="{ width: '100%' }">
          <a-progress
            :show-text="false"
            :steps="4"
            :color="item.color"
            :percent="item.bai"
            trackColor="rgb(var(--green-4))"
            stroke-width="30"
          />
        </div>
      </n-card>
    </n-grid-item>
  </n-grid>

  <n-divider
    dashed
    title-placement="center"
    style="font-size: 14px"
    v-if="itemslist.length < total && hasMoreData"
  >
    上拉加载更多
  </n-divider>

  <n-divider
    dashed
    title-placement="center"
    style="font-size: 14px"
    v-if="!hasMoreData"
  >
    没有更多数据了
  </n-divider>

  <n-back-top :right="20" :bottom="20" />
  <footer class="footer" v-html="footer"></footer>
</template>

<script>
import { defineComponent, ref, onMounted, onBeforeUnmount } from "vue";
import axios from "axios";
import { useLoadingBar } from "naive-ui";
import { Sparkles, SunnySharp } from "@vicons/ionicons5";
import { throttle } from "lodash-es"; // 使用ES模块版本

const isDev = import.meta.env.MODE === "development";

function uniqueArrayObjects(arr) {
  const jsonObjectSet = new Set();
  const uniqueArray = [];

  arr.forEach((item) => {
    const jsonString = JSON.stringify(item);
    if (!jsonObjectSet.has(jsonString)) {
      jsonObjectSet.add(jsonString);
      uniqueArray.push(item);
    }
  });

  return uniqueArray;
}

export default defineComponent({
  name: "HomePage",
  props: {
    isDarkTheme: {
      type: Boolean,
      default: false,
    },
  },
  setup(props, { emit }) {
    const itemslist = ref([]);
    const notice = ref("");
    const footer = ref("© 2025 玖亿科技 版权所有");
    const total = ref(0);
    const page = ref(1);
    const isLoading = ref(false);
    const hasMoreData = ref(true);
    const minPage = ref(10); // 保持原来的每页加载数量
    const loadingBar = ref(null);
    const statusCache = ref({}); // 状态缓存

    // 数据获取函数
    const fetchData = async () => {
      if (!hasMoreData.value || isLoading.value) return;

      if (!loadingBar.value) {
        loadingBar.value = useLoadingBar();
      }
      loadingBar.value.start();

      isLoading.value = true;

      try {
        const response = await axios.post(isDev ? "/api/carpage" : "/carpage", {
          page: page.value,
          size: minPage.value,
        });

        if (!response.data.data.list || response.data.data.list.length === 0) {
          hasMoreData.value = false;
          loadingBar.value.finish();
          isLoading.value = false;
          return;
        }

        notice.value = response.data.notice;
        total.value = response.data?.data?.pagination?.total ?? 0;
        page.value += 1;

        // 预处理数据并添加骨架屏状态
        const preprocessedList = response.data.data.list.map((item) => ({
          ...item,
          loading: true, // 初始加载中状态，显示骨架屏
          bai: 0,
          color: "#24d4ae",
          message: "加载中...",
        }));

        // 添加到列表
        itemslist.value = uniqueArrayObjects([
          ...itemslist.value,
          ...preprocessedList,
        ]);

        // 快速批量更新状态 - 使用Promise.all并行请求
        const updatePromises = preprocessedList.map((item) =>
          updateSingleItemStatus(item)
        );
        await Promise.all(updatePromises);
      } catch (error) {
        console.error("请求错误:", error);
        loadingBar.value.error();
      } finally {
        isLoading.value = false;
        loadingBar.value.finish();
      }
    };

    // 更新单个项目的状态 - 设置超时以加快响应速度
    const updateSingleItemStatus = async (item) => {
      try {
        const baseUrl = isDev ? "/api" : "";
        const carname = encodeURIComponent(`${item.carID}`);

        // 检查缓存是否有最近的数据
        const cacheKey = item.carID;
        const cachedData = statusCache.value[cacheKey];
        const now = Date.now();

        // 如果有缓存且不超过60秒，直接使用缓存
        if (cachedData && now - cachedData.timestamp < 60000) {
          updateItemInList(item.carID, {
            ...cachedData.data,
            loading: false, // 加载完成
          });
          return;
        }

        // 设置请求超时 - 最多等待2秒
        const timeout = 2000;

        // 创建带超时的fetch函数
        const fetchWithTimeout = (url, options = {}) => {
          return Promise.race([
            fetch(url, options),
            new Promise((_, reject) =>
              setTimeout(() => reject(new Error("Request timed out")), timeout)
            ),
          ]);
        };

        // 并行请求endpoint和status，加上超时控制
        try {
          const [endpointData, statusData] = await Promise.all([
            fetchWithTimeout(`${baseUrl}/endpoint?carid=${carname}`)
              .then((r) => r.json())
              .catch(() => ({})),
            fetchWithTimeout(`${baseUrl}/status?carid=${carname}`)
              .then((r) => r.json())
              .catch(() => ({})),
          ]);

          // 处理数据
          const processedEndpointData = {};
          for (let key in endpointData) {
            if (typeof endpointData[key] === "string") {
              processedEndpointData[key] = replaceStopRunning(
                endpointData[key]
              );
            } else {
              processedEndpointData[key] = endpointData[key];
            }
          }

          const bai = ((statusData.count || 0) / 100).toFixed(2);

          // 更新的数据
          const updatedData = {
            ...processedEndpointData,
            ...statusData,
            bai,
            loading: false, // 加载完成
          };

          // 更新缓存
          statusCache.value[cacheKey] = {
            data: updatedData,
            timestamp: now,
          };

          // 更新列表中的项目
          updateItemInList(item.carID, updatedData);
        } catch (error) {
          console.error(`Request timeout or error for ${item.carID}:`, error);
          // 如果超时，仍然更新状态以移除加载中
          updateItemInList(item.carID, {
            loading: false,
            message: "数据请求超时",
          });
        }
      } catch (error) {
        console.error(`Error updating item ${item.carID}:`, error);
        // 即使出错也要移除加载状态
        updateItemInList(item.carID, {
          loading: false,
          message: "数据获取失败",
        });
      }
    };

    // 更新列表中的单个项目 - 直接更新特定项
    const updateItemInList = (carID, newData) => {
      const index = itemslist.value.findIndex((item) => item.carID === carID);
      if (index !== -1) {
        // 使用对象扩展合并更新
        itemslist.value[index] = { ...itemslist.value[index], ...newData };
      }
    };

    // 辅助函数：替换状态文本
    const replaceStopRunning = (text) => {
      if (!text) return text;
      return text
        .replace("PLUS停运｜", "")
        .replace("TEAM停运｜", "")
        .replace("停运｜", "")
        .replace("|", "-")
        .replace("|", "-")
        .replace("PLUS", "Pro")
        .replace("3.7", "Free")
        .replace("green", "#f9bd5f")
        .replace("yellow", "#f9bd5f")
        .replace("red", "#f65e5d")
        .replace("blue", "#24d4ae")
        .replace("purple", "#a07be6");
    };

    // 主题样式
    const theme = () => {
      return props.isDarkTheme
        ? { "background-color": "#252529" }
        : { "background-color": "#eff4f9" };
    };

    // 跳转函数
    const redirectTo = (carID) => {
      window.location.href = `${
        window.location.origin
      }/auth/login?carid=${encodeURI(carID)}`;
    };

    // 滚动事件处理（使用节流）
    const handleScroll = () => {
      const scrollPosition = window.scrollY + window.innerHeight;
      const documentHeight = document.documentElement.scrollHeight;

      if (
        documentHeight - scrollPosition < 300 &&
        !isLoading.value &&
        hasMoreData.value
      ) {
        fetchData();
      }
    };

    const throttledHandleScroll = throttle(handleScroll, 200);

    // 页面可见性变化处理 - 当页面重新可见时更新数据
    const handleVisibilityChange = () => {
      if (!document.hidden && itemslist.value.length > 0) {
        // 页面变为可见时刷新前10个卡片数据
        const itemsToRefresh = itemslist.value.slice(0, 10);
        itemsToRefresh.forEach((item) => updateSingleItemStatus(item));
      }
    };

    // 生命周期钩子
    onMounted(() => {
      // 初始加载
      fetchData();

      // 添加滚动事件监听
      window.addEventListener("scroll", throttledHandleScroll);

      // 添加可见性变化事件监听
      document.addEventListener("visibilitychange", handleVisibilityChange);
    });

    onBeforeUnmount(() => {
      // 清理资源
      window.removeEventListener("scroll", throttledHandleScroll);
      document.removeEventListener("visibilitychange", handleVisibilityChange);
    });

    return {
      itemslist,
      notice,
      footer,
      total,
      hasMoreData,
      isLoading,
      Sparkles,
      SunnySharp,
      theme,
      redirectTo,
    };
  },
});
</script>

<style>
.n-button {
  border-radius: 7px;
  float: right;
}

.n-gradient-text {
  margin-top: 10px;
}

.n-card {
  border-radius: 5px;
  background-color: rgb(187 201 217 / 20%);
}

.type {
  margin-right: 5px;
  font-size: 12px;
  font-weight: 800;
  padding: 1px 6px;
  border-radius: 5px;
  color: #fff;
}

.message-with-dot {
  margin-top: 10px;
  margin-bottom: 5px;
  position: relative;
  color: #8f8f8f;
  font-size: 13px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.n-switch {
  float: right;
}

.n-badge .n-badge-sup {
  border-radius: 2px;
  padding: 10px 10px;
}

.cardclss {
  transition: transform 0.3s, box-shadow 0.3s;
  overflow: hidden;
  border-radius: 5px; /* 保持圆角 */
  cursor: pointer; /* 添加手型光标 */
}

.cardclss:hover {
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2); /* 悬停时更大的阴影 */
  transform: translateY(-3px); /* 上浮效果 */
}

.cardclss .title {
  font-weight: 600;
}

.notice {
  max-width: 100%;
  color: var(--n-title-text-color);
  border-radius: 10px !important;
  margin-bottom: 10px;
  background-color: rgb(187 201 217 / 20%) !important;
}

.arco-progress-steps-item {
  border-radius: 30px !important;
}

.plus-badge {
  position: relative; /* 确保伪元素能正确定位 */
  overflow: hidden; /* 防止动画溢出容器 */
}

.plus-badge::after {
  content: ""; /* 必须设置，否则伪元素不会显示 */
  position: absolute;
  top: 0;
  left: -100%; /* 开始时光条位于元素左侧外 */
  width: 100%; /* 光条宽度为整个元素宽度 */
  height: 100%; /* 光条高度为整个元素高度 */
  background: linear-gradient(
    to right,
    transparent,
    rgba(255, 255, 255, 0.8),
    transparent
  );
  animation: sweepLight 3s linear infinite; /* 调整时间和函数以符合需要 */
}

@keyframes sweepLight {
  0% {
    left: -100%; /* 动画开始位置 */
  }
  50% {
    left: 100%; /* 动画中间位置，光条完全覆盖元素 */
  }
  100% {
    left: 200%; /* 动画结束位置，光条移出元素右侧 */
  }
}

.box {
  display: flex;
  align-items: center; /* 确保子元素垂直居中 */
  justify-content: left; /* 子元素水平居中（如果需要） */
}
/* 禁止选择文字 */
.no-select {
  -webkit-user-select: none; /* Chrome, Safari, Opera */
  -moz-user-select: none; /* Firefox */
  -ms-user-select: none; /* Internet Explorer/Edge */
  user-select: none; /* Non-prefixed version, currently supported by Chrome, Edge, Opera and Firefox */
}
.footer {
  background-color: rgb(187 201 217 / 20%);
  color: var(--n-title-text-color);
  border-radius: 10px;
  text-align: center;
  padding: 10px 0;
  font-size: 14px;
  margin-top: 10px;
}

/* 完全自定义的卡片大小骨架屏样式 */
.card-skeleton {
  background-color: rgb(187 201 217 / 20%);
  border-radius: 5px;
  padding: 16px;
  height: 107px; /* 与卡片高度相同 */
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.skeleton-header {
  display: flex;
  align-items: center;
}

.skeleton-badge {
  width: 35px;
  height: 16px;
  background: #e0e0e0;
  border-radius: 5px;
  animation: skeleton-pulse 1.5s infinite ease-in-out;
}

.skeleton-title {
  height: 16px;
  width: 80px;
  margin-left: 10px;
  background: #e0e0e0;
  border-radius: 3px;
  animation: skeleton-pulse 1.5s infinite ease-in-out;
}

.skeleton-status {
  height: 14px;
  width: 80%;
  background: #e0e0e0;
  border-radius: 3px;
  margin: 10px 0;
  animation: skeleton-pulse 1.5s infinite ease-in-out;
}

.skeleton-progress {
  height: 14px;
  width: 100%;
  background: #e0e0e0;
  border-radius: 30px;
  animation: skeleton-pulse 1.5s infinite ease-in-out;
}

@keyframes skeleton-pulse {
  0% {
    opacity: 0.5;
  }
  50% {
    opacity: 0.8;
  }
  100% {
    opacity: 0.5;
  }
}
</style>