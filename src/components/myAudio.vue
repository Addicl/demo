<template>
    <audio ref="audioRef" :id="props.id" :src="props.audioUrl" @timeupdate="timeUpdate" @ended="onEnded" @loadedmetadata="onLoaded"></audio>
    <div class="audio-div">
        <div class="control" @click="controlAudio">
            <img :class="!playStatus ? 'play' : 'stop'" src="../assets/start.png" alt="">
            <img :class="playStatus ? 'play' : 'stop'" src="../assets/pause.png" alt="">
        </div>
        <div class="time">
            <div ref="progressBarRef" class="progress-bar" @click="progressBarClick">
                <div ref="progressRef" @mousedown="onMouseDown" class="progress" :style="{'left': progressPosition + 'px'}"></div>
            </div>
            <div class="timer">
                <span class="current-time">{{ currentTime.toFixed(2) }}</span> /
                <span class="total-time">{{ totalTime.toFixed(2) }}</span>
            </div>
        </div>
    </div>
</template>

<script lang="ts" setup>
import { ref, defineProps, onMounted, onUnmounted, watch } from 'vue';

const audioRef = ref<HTMLAudioElement>();
const progressBarRef = ref<HTMLDivElement>();
const progressRef = ref<HTMLDivElement>();
const playStatus = ref(true);
const currentTime = ref<number>(0);
const progressPosition = ref(0);
let totalTime = 0;
let offsetX = 0;
let rect: DOMRect | null = null;
let progressBarWidth = 0;

const props = defineProps({
    id: String,
    audioUrl: String,
});

// 播放控制
const controlAudio = () => {
    if (playStatus.value) {
        audioRef.value?.play();
    } else {
        audioRef.value?.pause();
    }
    playStatus.value = !playStatus.value;
};

// 播放结束改变状态
const onEnded = () => {
    currentTime.value = 0;
    playStatus.value = true;
};

// 加载一遍获取总时长
const onLoaded = () => {
    totalTime = audioRef.value?.duration || 0;
    if (totalTime === Infinity) {
        audioRef.value!.currentTime = 1e101;
        audioRef.value!.ontimeupdate = function () {
            totalTime = audioRef.value?.duration || 0;
            if (totalTime !== Infinity) {
                audioRef.value!.ontimeupdate = null;
                audioRef.value!.currentTime = 0;
            }
        };
    }
};

// 播放时间更新
const timeUpdate = () => {
    if (audioRef.value) {
        currentTime.value = audioRef.value.currentTime;
    }
};

// 监听当前时间变化并更新进度条位置
watch(currentTime, (newValue) => {
    progressPosition.value = newValue / totalTime * 200;
});

// 进度条点击跳转
const progressBarClick = (e: MouseEvent) => {
    if (audioRef.value && progressBarRef.value) {
        const progressBarWidth = progressBarRef.value.offsetWidth;
        const rect = progressBarRef.value.getBoundingClientRect();
        progressPosition.value = e.clientX - rect.left;
        const currentTime = ((e.clientX - rect.left) / progressBarWidth) * totalTime;
        audioRef.value.currentTime = currentTime;
    }
};

const onMouseDown = (e: MouseEvent) => {
    if (!progressBarRef.value || !progressRef.value) return;
    // 缓存进度条的位置和宽度
    rect = progressBarRef.value.getBoundingClientRect();
    progressBarWidth = progressBarRef.value.offsetWidth;
    // 记录鼠标相对进度条的位置
    offsetX = e.offsetX - rect!.left;
    document.addEventListener('mousemove', onMouseMove);
    document.addEventListener('mouseup', onMouseUp);
};

const onMouseMove = (e: MouseEvent) => {
    if (!rect || !progressBarWidth || !audioRef.value) return;
    // 计算当前鼠标位置对应的进度
    const moveDistance = e.clientX - rect.left - offsetX;
    progressPosition.value = Math.max(0, Math.min(moveDistance + offsetX, progressBarWidth));
    const currentTime = (progressPosition.value / progressBarWidth) * totalTime;
    audioRef.value.currentTime = currentTime;
};

const onMouseUp = () => {
    document.removeEventListener('mousemove', onMouseMove);
    document.removeEventListener('mouseup', onMouseUp);
};

// 生命周期钩子
onMounted(() => {
    if (progressRef.value) {
        progressRef.value.addEventListener('mousedown', onMouseDown);
    }
});

onUnmounted(() => {
    if (progressRef.value) {
        progressRef.value.removeEventListener('mousedown', onMouseDown);
    }
});
</script>

<style lang="less" scoped>
.audio-div {
    position: relative;
    width: 300px;
    height: 65px;
    border-radius: 25px;
    background-color: #f5f5f5;
    .control {
        position: relative;
        width: 45px;
        height: 45px;
        img {
            position: absolute;
            top: 10px;
            left: 5px;
            width: 100%;
        }
    }
    .time {
        position: absolute;
        top: 20px;
        left: 60px;
        .progress-bar {
            width: 200px;
            height: 25px;
            background-color: rgba(200, 200, 200, 0.3);
            position: relative;
            .progress {
                position: absolute;
                height: 100%;
                width: 5px;
                background-color: black;
                cursor: pointer;
            }
        }
        .timer {
            position: absolute;
            right: 70px;
        }
    }
}
.play {
    opacity: 0;
}
.stop {
    opacity: 1;
}
</style>