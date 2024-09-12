<template>
  <div class="my-record">
      <button class="start" @click="controlRecord">{{ btnText }}</button>
      <!-- <button class="download" @click="downloadRecord">下载</button> -->
      <button class="reStart" @click="reStart">重新录制</button>
      <button @click="downloadRecord">导出音频</button>
  </div>
  <p>{{ timer }}s</p>
  <MyAudio :id="'audio-part'" :audioUrl="audioBlobUrl" :time="time"></MyAudio>
  <!-- <AudioPart :id="'audio-part'" :audioUrl="audioBlobUrl"></AudioPart> -->
  <!-- <audio ref="audioRef" ></audio> -->
 
</template>

<script lang="ts" setup>
import { computed, ref } from 'vue';
import MyAudio from './components/myAudio.vue';
const mediaStream = ref<MediaStream | null>(null); // 用于存储用户媒体流
const audioChunks = ref<Blob[]>([]);  // 用于存储录制的音频片段
const mediaRecorder = ref<MediaRecorder | null>(null);  // 用于存储录制器实例
const audioBlobUrl = ref('');  // 用于存储音频的URL
const timer = ref<number>(60);  // 用于存储计时器
const btnText = ref('开始录音');  // 用于存储按钮文字
const isRecording = ref(false);  // 用于存储录音状态
let countdownInterval: number | null = null;  // 用于存储计时器
const timeStart = ref<number>();
const timeEnd = ref<number>();
const  time = computed(() => {
  if (timeStart.value && timeEnd.value) {
    return (timeEnd.value - timeStart.value)/1000;
  } else {
    return 0;
  }
});
// 获取用户媒体
const getUserMedia = (constraints: MediaStreamConstraints): Promise<MediaStream> => {
  if (navigator.mediaDevices === undefined) {
      (navigator as any).mediaDevices = {};
  }
  const legacyGetUserMedia =
      (navigator as any).getUserMedia ||
      (navigator as any).webkitGetUserMedia ||
      (navigator as any).mozGetUserMedia;
  if (!navigator.mediaDevices.getUserMedia && legacyGetUserMedia) {
      navigator.mediaDevices.getUserMedia = function(constraints: MediaStreamConstraints): Promise<MediaStream> {
          return new Promise((resolve, reject) => {
              legacyGetUserMedia.call(navigator, constraints, resolve, reject);
          });
      };
  }
  return navigator.mediaDevices.getUserMedia(constraints);
};
// 控制录音
const controlRecord = () => {
  if (!isRecording.value) {
      isRecording.value = true;
      btnText.value = '停止录音';
      startRecord();
      startCountdown();
  } else {
      isRecording.value = false;
      btnText.value = '开始录音';
      stopRecord();
      stopCountdown();
  }
};
//开始录音
const startRecord = async () => {
  try {
    mediaStream.value = await getUserMedia({ audio: true });
    mediaRecorder.value = new MediaRecorder(mediaStream.value);
    audioChunks.value = [];
    mediaRecorder.value.ondataavailable = (event) => {
      if (event.data.size > 0) {
          audioChunks.value.push(event.data);
      }
      playAudio();
    };
    mediaRecorder.value.start();
    timeStart.value=Date.now();
  } catch (error) {
      console.error('获取麦克风权限失败:', error);
  }
};
//停止录音
const stopRecord = () => {
  if (mediaRecorder.value) {
      timeEnd.value=Date.now();
      mediaRecorder.value.stop();
  } else {
      console.warn('mediaRecorder.value is null, cannot call stop method.');
  }
};
//开始倒计时
const startCountdown = () => {
  countdownInterval = window.setInterval(() => {
      if (timer.value > 0) {
          timer.value--;
      } else {
          // 当倒计时结束时自动停止录音
          stopRecord();
          stopCountdown();
      }
  }, 1000);
};
//停止倒计时
const stopCountdown = () => {
  if (countdownInterval !== null) {
      window.clearInterval(countdownInterval);
      countdownInterval = null;
  }
};
//重新录制
const reStart = () => {
  audioChunks.value = [];
  audioBlobUrl.value = '';
  timer.value = 60;
  controlRecord();
};
//生成音频
const playAudio = () => {
  if ( audioChunks.value.length > 0) {
      const audioBlob = new Blob(audioChunks.value, { type: 'audio/webm' });
      audioBlobUrl.value = URL.createObjectURL(audioBlob);
  } else {
      console.warn('没有录音数据可播放');
  }
};
//导出音频
const downloadRecord = () => {
  if (audioBlobUrl.value) {
      const link = document.createElement('a');
      link.href = audioBlobUrl.value;
      link.download = 'record.webm';
      link.click();
  } else {
      console.warn('没有录音数据可下载');
  }
};
//上传到指定服务器
const uploadRecord = async () => {
  const formData = new FormData();
  formData.append('file', audioChunks.value[0]);
  const response = await fetch('http://localhost:3000/upload', {
      method: 'POST',
      body: formData
  });
  const data = await response.json();
  console.log(data);
};
</script>

<style lang="less" scoped>
.my-record {
  display: flex;
  flex-direction: row;
  justify-content: left;
}
</style>