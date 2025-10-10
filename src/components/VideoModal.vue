<template>
  <NModal
    v-model:show="show"
    style="width: 70%"
    title="视频播放"
    preset="card"
    :bordered="false"
    :draggable="true"
    @after-leave="handleVideoClose"
  >
    <NLayout has-sider :content-style="layoutHeightStyle">
      <NLayoutSider :native-scrollbar="false" bordered>
        <!-- 筛选器区域 -->
        <div class="filter-section">
          <NInput
            v-model:value="searchText"
            placeholder="搜索视频..."
            size="small"
            clearable
            class="search-input"
          >
            <template #prefix>
              <NIcon><SearchOutlined /></NIcon>
            </template>
          </NInput>

          <div class="sort-controls">
            <NSelect
              v-model:value="sortBy"
              :options="sortOptions"
              size="small"
              placeholder="排序方式"
              class="sort-select"
            />
          </div>

          <!-- <NCheckbox v-model:checked="hideWatched" size="small" class="hide-watched-checkbox">
            隐藏已观看
          </NCheckbox> -->
        </div>

        <NList hoverable clickable>
          <NListItem
            v-for="(video, index) in filteredVideoList"
            :key="video.code"
            :class="{ 'selected-video': currentVideoCode === video.code }"
            @click="handleVideoClick(video)"
          >
            <NThing content-style="margin-top: 10px;">
              <template #description>
                <NSpace size="small" style="margin-top: 4px">
                  <NTag :bordered="false" type="info" size="small"> # {{ index + 1 }} </NTag>
                  <NTag v-if="video.size" :bordered="false" type="success" size="small">
                    {{ formatFileSize(video.size || 0) }}
                  </NTag>
                  <NTooltip v-if="video.playFailed" trigger="hover">
                    <template #trigger>
                      <NTag :bordered="false" type="error" size="small">错误</NTag>
                    </template>
                    {{ video.error || '视频播放失败' }}
                  </NTooltip>
                </NSpace>
              </template>

              <NEllipsis style="max-width: 100%">
                {{ video.name }}
              </NEllipsis>
            </NThing>
          </NListItem>
        </NList>
      </NLayoutSider>
      <NLayout :native-scrollbar="false">
        <div ref="videoRef" class="video-js"></div>
      </NLayout>
    </NLayout>
  </NModal>
</template>

<script setup lang="ts">
  import Player, { type IPlayerOptions } from 'xgplayer';
  import 'xgplayer/dist/index.min.css';
  import HlsJsPlugin from 'xgplayer-hls.js';
  import { Events } from 'xgplayer';
  import { SearchOutlined } from '@vicons/antd';
  import { request, settings, formatFileSize } from '@/utils';

  /** 视频项接口定义 */
  interface VideoItem {
    name: string;
    code: string;
    size?: number;
    suffix?: string;
    url?: string;
    time?: number;
    playFailed?: boolean;
    error?: string;
  }

  /** API响应类型 */
  interface VideoApiResponse {
    state: boolean;
    video_url?: string;
    error?: string;
  }

  interface HistoryApiResponse {
    state: boolean;
    data: { time?: number };
    error?: string;
  }

  // Props 和 Model 定义
  const show = defineModel('show', {
    type: Boolean,
    default: false,
  });

  const props = defineProps<{
    data: VideoItem[];
  }>();

  // 组合式 API
  const message = useMessage();

  // 响应式数据
  const videoList = ref<VideoItem[]>([]);
  const videoRef = useTemplateRef('videoRef');
  const player = ref<Player | null>(null);
  const layoutHeight = ref<number>(700);
  const currentVideo = ref<VideoItem | null>(null);
  const currentVideoCode = ref<string>('');

  // 筛选相关的响应式数据
  const searchText = ref<string>('');
  const sortBy = ref<string>('name-asc');

  // 排序选项
  const sortOptions = [
    { label: '按名称排序 △', value: 'name-asc' },
    { label: '按名称排序 ▽', value: 'name-desc' },
    { label: '按添加顺序 △', value: 'order-asc' },
    { label: '按添加顺序 ▽', value: 'order-desc' },
    { label: '按文件大小 △', value: 'size-asc' },
    { label: '按文件大小 ▽', value: 'size-desc' },
  ];

  // 使用 VueUse 的 useIntervalFn 来管理定时保存播放进度
  const { pause: pauseSaveTimer, resume: resumeSaveTimer } = useIntervalFn(
    () => {
      if (!player.value || !currentVideo.value || player.value.paused) return;
      if (!videoSettings.value.enableHistory) return;

      const currentTime = Math.floor(player.value.currentTime);
      if (currentTime && currentTime !== currentVideo.value.time) {
        currentVideo.value.time = currentTime;
        setVideoHistory(currentVideo.value.code, currentTime);
      }
    },
    5000,
    { immediate: false },
  );

  // 计算属性
  const layoutHeightStyle = computed(() => ({
    height: `${layoutHeight.value}px`,
  }));

  const videoSettings = computed(() => ({
    autoplay: settings?.video.autoplay ?? true,
    volume: settings?.video.volume ?? 1,
    defaultPlaybackRate: settings?.video.defaultPlaybackRate ?? 1,
    enableHistory: settings?.video.history ?? true,
    autoNext: settings?.video.autoNext ?? true,
    autoSkipFailed: settings?.video.autoSkipFailed ?? true,
  }));

  // 筛选后的视频列表
  const filteredVideoList = computed(() => {
    let filteredVideos = [...videoList.value];

    // 按搜索文本筛选
    if (searchText.value.trim()) {
      const searchTerm = searchText.value.toLowerCase().trim();
      filteredVideos = filteredVideos.filter((video) =>
        video.name.toLowerCase().includes(searchTerm),
      );
    }

    // 解析排序方式和方向
    const [sortType, sortDirection] = sortBy.value.split('-') as [string, 'asc' | 'desc'];

    // 排序
    switch (sortType) {
      case 'name':
        filteredVideos.sort((a, b) => {
          const result = a.name.localeCompare(b.name);
          return sortDirection === 'asc' ? result : -result;
        });
        break;
      case 'size':
        // 按实际文件大小排序
        filteredVideos.sort((a, b) => {
          const sizeA = a.size || 0;
          const sizeB = b.size || 0;
          const result = sizeA - sizeB;
          return sortDirection === 'asc' ? result : -result;
        });
        break;
      case 'order':
      default:
        // 保持原有顺序或反转
        if (sortDirection === 'desc') {
          filteredVideos.reverse();
        }
        break;
    }

    return filteredVideos;
  });

  // 监听器
  watch(show, (value) => {
    if (value) {
      initializeVideoList();
    }
  });

  // 工具函数
  const stopSaveTimer = (): void => {
    pauseSaveTimer();
    currentVideo.value = null;
  };

  const destroyPlayer = (): void => {
    if (player.value) {
      player.value.destroy();
      player.value = null;
    }
  };

  /**
   * 判断视频是否接近播放完成
   * @param currentTime 当前播放时间（秒）
   * @param duration 视频总时长（秒）
   * @param threshold 阈值（秒），默认为30秒
   * @returns 是否接近播放完成
   */
  const isVideoNearlyCompleted = (
    currentTime: number,
    duration: number,
    threshold: number = 30,
  ): boolean => {
    if (duration <= 0 || currentTime <= 0) return false;

    // 如果视频总时长小于阈值，则认为播放到80%以上就算接近完成
    if (duration < threshold) {
      return currentTime / duration >= 0.8;
    }

    // 如果剩余时间小于阈值，则认为接近完成
    return duration - currentTime <= threshold;
  };

  /**
   * 初始化视频列表
   */
  const initializeVideoList = (): void => {
    videoList.value = [...props.data];
    play();
  };

  /**
   * 开始播放第一个可用的视频
   */
  const play = async (): Promise<void> => {
    try {
      if (filteredVideoList.value.length === 0) {
        throw new Error('视频列表为空');
      }

      // 尝试播放第一个可用的视频
      let playSuccess = false;
      for (let i = 0; i < filteredVideoList.value.length; i++) {
        const video = filteredVideoList.value[i];
        if (!video) continue;

        try {
          await setupVideoForPlay(video);
          await createPlayer(video);
          playSuccess = true;
          break;
        } catch (error) {
          console.warn(`视频 ${video.name} 播放失败:`, error);

          player.value?.destroy();
          player.value = null;

          // 如果不自动跳过失败视频，直接抛出错误
          if (!videoSettings.value.autoSkipFailed) {
            throw error;
          }

          // 如果不是最后一个视频，继续尝试下一个
          if (i < filteredVideoList.value.length - 1) {
            message.warning(`视频 ${video.name} 播放失败，正在尝试下一个视频`);
            continue;
          }
          // 如果是最后一个视频也失败了，抛出错误
          throw error;
        }
      }

      if (!playSuccess) {
        throw new Error('所有视频都无法播放');
      }
    } catch (error) {
      handleError('视频播放失败', error);
    }
  };

  /**
   * 检查播放器是否可用，如果不可用则重新初始化
   */
  const ensurePlayerExists = async (video: VideoItem): Promise<void> => {
    if (!player.value || !videoRef.value) {
      await createPlayer(video);
    }
  };

  /**
   * 设置视频播放所需数据
   */
  const setupVideoForPlay = async (video: VideoItem): Promise<void> => {
    currentVideoCode.value = video.code;

    try {
      video.url = await getVideoUrl(video.code);
      video.time = videoSettings.value.enableHistory ? (await getVideoHistory(video.code)) || 0 : 0;
      // 清除之前的错误状态
      video.playFailed = false;
      video.error = undefined;
    } catch (error) {
      // 设置失败状态
      video.playFailed = true;
      video.error = error instanceof Error ? error.message : String(error);
      throw error; // 依然抛出错误，供上层处理
    }
  };

  /**
   * 创建播放器实例
   */
  const createPlayer = async (video: VideoItem): Promise<void> => {
    if (!videoRef.value || !video.url) {
      throw new Error('播放器容器或视频URL不可用');
    }

    const baseConfig: IPlayerOptions = {
      el: videoRef.value,
      url: video.url,
      autoplay: videoSettings.value.autoplay,
      fluid: true,
      volume: videoSettings.value.volume,
      defaultPlaybackRate: videoSettings.value.defaultPlaybackRate,
      playbackRate: { list: [5, 4, 3, 2, 1.5, 1.25, 1, 0.75, 0.5, 0.25, 0.1] },
      rotate: true,
      pip: true,
      dynamicBg: { disable: false },
    };

    // 根据浏览器支持情况选择播放器配置
    const canPlayHLS = document.createElement('video').canPlayType('application/vnd.apple.mpegurl');

    if (canPlayHLS) {
      player.value = new Player(baseConfig);
    } else if (HlsJsPlugin.isSupported()) {
      player.value = new Player({
        ...baseConfig,
        isLive: false,
        plugins: [HlsJsPlugin],
      });
    } else {
      throw new Error('浏览器不支持HLS播放');
    }

    setupPlayerEvents(video);
  };

  /**
   * 设置播放器事件
   */
  const setupPlayerEvents = (video: VideoItem): void => {
    if (!player.value) return;

    // 监听视频尺寸变化
    player.value.on(Events.VIDEO_RESIZE, () => {
      layoutHeight.value = videoRef.value?.clientHeight || 700;
    });

    // 监听视频播放结束事件
    player.value.on(Events.ENDED, () => {
      handleVideoEnded();
    });

    // 监听视频元数据加载完成事件，设置历史播放位置
    if (videoSettings.value.enableHistory && video.time && video.time > 1) {
      player.value.once(Events.LOADED_METADATA, () => {
        if (!player.value) return;

        const duration = player.value.duration;
        const isNearlyCompleted = isVideoNearlyCompleted(video.time || 0, duration);

        if (!isNearlyCompleted) {
          // 如果视频没有接近播放完成，则从历史位置开始播放
          player.value.currentTime = video.time || 0;

          // 提示用户从历史播放位置开始播放
          // TODO 如果之前的视频失败，点击播放正常视频时，会有两个提示
          // const timeText = formatTime(video.time || 0);
          // message.info(`从上次播放位置开始：${timeText}`);
        } else {
          // 如果接近播放完成，则从头开始播放，给出提示
          message.info('视频已接近播放完成，从头开始播放');
        }
      });
    }

    // 启动保存计时器
    if (videoSettings.value.enableHistory) {
      startSaveTimer(video);
    }
  };

  /**
   * 启动定时保存播放进度
   */
  const startSaveTimer = (video: VideoItem): void => {
    stopSaveTimer();

    if (!videoSettings.value.enableHistory || !player.value) return;

    currentVideo.value = video;
    resumeSaveTimer();
  };

  /**
   * 获取视频播放地址
   */
  const getVideoUrl = async (code: string): Promise<string> => {
    const res = await request({
      method: 'GET',
      url: `https://webapi.115.com/files/video?pickcode=${code}&share_id=0&local=1`,
    });

    if (res.status !== 200) {
      throw new Error('获取视频地址失败');
    }

    const json: VideoApiResponse = JSON.parse(res.responseText);
    if (!json.state) {
      throw new Error(json.error || '获取视频地址失败');
    }

    if (!json.video_url) {
      throw new Error('视频地址获取失败');
    }

    return json.video_url.replace('http://', 'https://');
  };

  /**
   * 获取视频播放历史
   */
  const getVideoHistory = async (code: string): Promise<number> => {
    const res = await request({
      method: 'GET',
      url: `https://webapi.115.com/files/history?pick_code=${code}&fetch=one&category=1&share_id=0`,
    });

    const json: HistoryApiResponse = JSON.parse(res.responseText);
    if (!json.state) {
      if (json.error) {
        throw new Error(json.error);
      }
      return 0;
    }

    return json.data?.time || 0;
  };

  /**
   * 设置视频播放历史
   */
  const setVideoHistory = async (code: string, time: number): Promise<void> => {
    request({
      method: 'POST',
      url: 'https://webapi.115.com/files/history',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      data: `op=update&pick_code=${code}&time=${time}&definition=0&category=1&share_id=0`,
    });
  };

  /**
   * 处理视频项点击
   */
  const handleVideoClick = async (video: VideoItem): Promise<void> => {
    try {
      // 如果视频之前失败过，重新尝试
      if (video.playFailed) {
        message.info(`重新尝试播放视频：${video.name}`);
      }

      currentVideoCode.value = video.code;

      // 如果视频URL未缓存或之前失败过，则重新获取
      if (!video.url || video.playFailed) {
        await setupVideoForPlay(video);
      }

      // 确保播放器存在，如果不存在则重新创建
      await ensurePlayerExists(video);

      // 用户主动点击视频时，使用历史播放时间（会自动判断是否接近播放完成）
      await switchVideo(video);
    } catch (error) {
      player.value?.destroy();
      player.value = null;

      handleError('视频切换失败', error);
    }
  };

  /**
   * 处理错误信息
   */
  const handleError = (title: string, error: unknown): void => {
    console.error(error);
    const errorMessage = error instanceof Error ? error.message : String(error);
    message.error(`${title}，错误信息：${errorMessage}`);
  };

  /**
   * 处理视频关闭
   */
  const handleVideoClose = (): void => {
    stopSaveTimer();
    destroyPlayer();
    show.value = false;
  };

  /**
   * 获取下一个视频
   */
  const getNextVideo = (): VideoItem | null => {
    const currentIndex = filteredVideoList.value.findIndex(
      (video) => video.code === currentVideoCode.value,
    );

    if (currentIndex === -1 || currentIndex >= filteredVideoList.value.length - 1) {
      return null; // 没有下一个视频
    }

    return filteredVideoList.value[currentIndex + 1] || null;
  };

  /**
   * 自动播放下一个视频（由视频播放结束触发）
   */
  const autoPlayNextVideo = async (video: VideoItem): Promise<void> => {
    try {
      currentVideoCode.value = video.code;

      // 如果视频URL未缓存或之前失败过，则重新获取
      if (!video.url || video.playFailed) {
        await setupVideoForPlay(video);
      }

      // 确保播放器存在，如果不存在则重新创建
      await ensurePlayerExists(video);

      // 自动播放下一个视频时，使用历史播放时间（会自动判断是否接近播放完成）
      await switchVideo(video);
    } catch (error) {
      // 如果自动播放失败且开启了自动跳过，尝试播放下一个
      if (videoSettings.value.autoSkipFailed) {
        const nextVideo = getNextVideo();
        if (nextVideo) {
          message.warning(`视频 ${video.name} 播放失败，正在尝试下一个视频`);
          await autoPlayNextVideo(nextVideo);
          return;
        }
      }
      handleError('自动播放视频失败', error);
    }
  };

  /**
   * 处理视频播放结束
   */
  const handleVideoEnded = async (): Promise<void> => {
    try {
      // 检查是否启用自动播放下一个视频
      if (!videoSettings.value.autoNext) {
        message.info('视频播放完成');
        return;
      }

      const nextVideo = getNextVideo();

      if (nextVideo) {
        message.info(`正在播放下一个视频：${nextVideo.name}`);
        await autoPlayNextVideo(nextVideo);
      } else {
        message.success('所有视频播放完成');
      }
    } catch (error) {
      handleError('自动播放下一个视频失败', error);
    }
  };

  /**
   * 格式化时间为 MM:SS 或 HH:MM:SS 格式
   */
  const formatTime = (seconds: number): string => {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = Math.floor(seconds % 60);

    if (hours > 0) {
      return `${hours}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
    }
    return `${minutes}:${secs.toString().padStart(2, '0')}`;
  };

  /**
   * 切换视频
   * @param video 要切换到的视频
   * @param useHistoryTime 是否使用历史播放时间，默认为true
   */
  const switchVideo = async (video: VideoItem, useHistoryTime: boolean = true): Promise<void> => {
    if (!player.value || !video.url) return;

    stopSaveTimer();

    player.value.src = video.url;

    if (videoSettings.value.autoplay) {
      player.value.play();
    }

    // 判断是否需要设置历史播放时间
    if (useHistoryTime && videoSettings.value.enableHistory && video.time && video.time > 1) {
      // 等待视频元数据加载完成后再判断是否接近播放完成
      player.value.once(Events.LOADED_METADATA, () => {
        if (!player.value) return;

        const duration = player.value.duration;
        const isNearlyCompleted = isVideoNearlyCompleted(video.time || 0, duration);

        if (!isNearlyCompleted) {
          // 如果视频没有接近播放完成，则从历史位置开始播放
          player.value.currentTime = video.time || 0;

          // 提示用户从历史播放位置开始播放
          const timeText = formatTime(video.time || 0);
          message.info(`从上次播放位置开始：${timeText}`);
        } else {
          // 如果接近播放完成，则从头开始播放，给出提示
          message.info('视频已接近播放完成，从头开始播放');
        }

        startSaveTimer(video);
      });
    } else if (videoSettings.value.enableHistory) {
      // 即使不使用历史时间，也要启动保存计时器
      startSaveTimer(video);
    }
  };
</script>

<style lang="scss" scoped>
  .filter-section {
    position: sticky;
    top: 0;
    z-index: 10;
    padding: 8px 8px 1px 8px;
    background-color: var(--n-color);

    .search-input {
      margin-bottom: 8px;
    }

    .sort-controls {
      display: flex;
      gap: 4px;
      margin-bottom: 8px;
      align-items: center;

      .sort-select {
        flex: 1;
      }
    }

    .n-input,
    .n-select {
      font-size: 12px;
    }
  }

  :deep(.n-list) {
    padding: 4px;
  }

  /* NList项的悬停效果 */
  :deep(.n-list .n-list-item) {
    padding: 8px 12px;
    border-radius: 4px;
    margin: 2px 0;
    transition: background-color 0.2s ease;
    cursor: pointer;

    &:hover {
      background-color: var(--n-color-hover) !important;
    }

    /* 选中的视频项样式 */
    &.selected-video {
      background-color: var(--n-color) !important;
      border-radius: 4px;
      color: #f40;

      .n-thing-header__title {
        color: var(--n-primary-color) !important;
        font-weight: 600;
      }
    }

    .n-list-item__main {
      width: 100%;
    }
    .n-thing-main {
      width: 100%;
    }
  }
</style>
