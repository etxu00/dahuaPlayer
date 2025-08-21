<script lang="ts" setup>
import { onMounted } from "vue";
// imouPlayer is provided globally by imou-player.js
declare const imouPlayer: any;

interface IPlayer {
  play: Function;
  stop: Function;
  pause: Function;
  capture: Function;
  startTalk: Function;
  stopTalk: Function;
  volume: Function;
  fullScreen: Function;
  exitFullScreen: Function;
  startRecord: Function;
  stopRecord: Function;
  destroy: Function;
}

let player: IPlayer;

const play = () => {
  player.play();
};
const pause = () => {
  player.pause();
};
const stop = () => {
  player.stop();
};
const capture = () => {
  player.capture();
};
const startTalk = () => {
  player.startTalk();
};
const stopTalk = () => {
  player.stopTalk();
};
const volume = (value: number) => {
  player.volume(value);
};
const fullScreen = () => {
  player.fullScreen();
};
const exitFullScreen = () => {
  player.exitFullScreen();
};
const startRecord = () => {
  player.startRecord();
};
const stopRecord = () => {
  player.stopRecord();
};

const destroy = () => {
  player.destroy();
  player = null!;
};

const init = () => {
  if (player) {
    destroy();
  }
  player = new imouPlayer({
    id: "imou-player",
    width: 1200,
    height: 700,
    domain: "https://openapi-or.easy4ip.com",
    deviceId: "AK05419PAZ99E87",
    channelId: "7",
    token: "Kt_or21c9258258a64ef1b773e2bf056003",
    // 1-Live 直播; 2-Playback 录播
    type: 1,
    // Live 0-HD 高清; 1-SD 标清
    streamId: 0,
    // 录播 云录像 cloud 本地录像 localRecord 默认 云录像
    // Playback, cloud-Cloud Video; localRecord-Local Video. Default Cloud Video
    recordType: "localRecord",     // or "localRecord"
    //beginTime: "2025-08-07 10:00:00",
    // endTime:   "2025-08-07 10:30:00",
    muted: false,
    code: "AK05419PAZ99E87",
    handleError: (err: unknown) => {
      console.error("handleError", err);
    },
  });
  window.player = player;
};

// --- Simple 2x2 grid test with the same channel/token ---
const gridIds = ["cell-0", "cell-1", "cell-2", "cell-3"];
let gridPlayers: any[] = [];

const destroyGrid = () => {
  gridPlayers.forEach(p => {
    try { p && p.destroy && p.destroy(); } catch {}
  });
  gridPlayers = [];
};

const initGrid = () => {
  destroyGrid();
  
  // Add delay between player initializations to avoid conflicts
  gridIds.forEach((id, index) => {
    setTimeout(() => {
      const el = document.getElementById(id) as HTMLElement;
      if (!el) return;
      
      // Use larger dimensions to avoid scaling issues
      const p = new imouPlayer({
        id,
        width: 640,
        height: 360,
        domain: "https://openapi-or.easy4ip.com",
        deviceId: "AK05419PAZ99E87",
        channelId: "7",
        token: "Kt_ore91a8d8f45db435b93ac297c260bcf",
        type: 1, // Live
        streamId: 1, // Use SD (1) instead of HD (0) for better performance with multiple streams
        recordType: "cloud", 
        muted: true,
        code: "AK05419PAZ99E87",
        handleError: (err: unknown) => console.error(id, err),
      });
      gridPlayers.push(p);
    }, index * 500); // 500ms delay between each player
  });
};

onMounted(() => {
  // init();
});
</script>

<template>
  <div class="imou-player">
    <div
      id="imou-player"
      style="width: 1200px; height: 700px; background-color: #000"
    ></div>
    <div>
      <button @click="init">Init imouPlayer</button>
      <button @click="play">Play</button>
      <button @click="pause">Pause</button>
      <button @click="stop">Stop</button>
      <button @click="capture">Capture</button>
      <button @click="startTalk">Start Talk</button>
      <button @click="stopTalk">Stop Talk</button>
      <button @click="() => volume(1)">Open Volume</button>
      <button @click="() => volume(0)">Close Volume</button>
      <button @click="fullScreen">FullScreen</button>
      <button @click="exitFullScreen">Exit FullScreen</button>
      <button @click="startRecord">Start Screen Recording</button>
      <button @click="stopRecord">Stop Screen Recording</button>
      <button @click="initGrid">Init Grid x4 (same channel)</button>
    </div>

    <!-- Simple 2x2 grid containers -->
    <div class="player-grid">
      <div v-for="id in gridIds" :key="id" :id="id" class="player-cell"></div>
    </div>
  </div>
</template>

<style scoped>
.player-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
  margin-top: 12px;
}

.player-cell {
  width: 100%;
  height: 360px; /* Match player height */
  background: #000;
  position: relative;
  overflow: hidden;
}

.player-cell canvas {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  max-width: 100%;
  max-height: 100%;
  width: auto;
  height: auto;
}
</style>
