<script lang="ts" setup>
import { ref, onMounted } from "vue";
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
const hasError = ref(false); // Variable reactiva para manejar el estado de error

const play              = () => { player.play() };
const pause             = () => { player.pause() };
const stop              = () => { player.stop() };
const capture           = () => { player.capture() };
const startTalk         = () => { player.startTalk() };
const stopTalk          = () => { player.stopTalk() };
const volume            = (value: number) => { player.volume(value) };
const fullScreen        = () => { player.fullScreen() };
const exitFullScreen    = () => { player.exitFullScreen() };
const startRecord       = () => { player.startRecord() };
const stopRecord        = () => { player.stopRecord() };
const destroy = () => {
  player.destroy();
  player = null!;
};
const getURLparams = () => {
  // Example .../index.html?token=Kt_or21c9258258a64ef1b773e2bf056003&noSerie=AK05419PAZ99E87
  // Se utiliza token en la URL para no indicar que es una contraseña. // TODO: encriptar datos

  const params: Record<string, string> = {};
  const urlParams = new URLSearchParams(window.location.search);
  urlParams.forEach((value, key) => {
    params[key] = value;
  });
  return {
    dvrPass: params.token || "",
    noSerie: params.noSerie || "",
  };
};

const verifyToken = (token: string): boolean => {
  // Simple token verification logic
  return typeof token === "string" && token.length > 0;
};

/**
 * Genera un fetch mediante una promise para obtener un token Bearer y lo guarda en   
 * localStorage como 'token_bear'.
 */
async function generateTokenBear(): Promise<string> {
  return new Promise((resolve, reject) => {
    const username = import.meta.env.VITE_API_USERNAME;
    const password = import.meta.env.VITE_API_PASSWORD;
    const url = import.meta.env.VITE_API_URL_TOKEN_BEAR
    fetch(url, {
      method: 'POST',
      headers: {
        'accept': 'application/json',
        'Content-Type': 'application/json-patch+json',
      },
      body: JSON.stringify({
        username: username,
        password: password,
        rememberMe: true
      }),
    })
    .then(response => {
      if (!response.ok) {
        return response.text().then((text) => {
          reject(`Error ${response.status}: ${text}`);
        });
      }
      return response.json();
    })
    .then(response => {
      if (!response?.accessToken) {
        reject('Invalid response from server');
      }

      const _tokenBear = response?.accessToken?.token;
      const dateExpire = response?.accessToken?.expiresIn;

      const expirationDate = new Date(Date.now() + dateExpire * 1000);

      localStorage.setItem('token_bear_expiration', expirationDate.toISOString());
      localStorage.setItem('token_bear', _tokenBear);

      resolve(_tokenBear);
    })
    .catch(error => {
      reject(`Network error: ${error.message}`);
    });
  });
}

async function generateTokenimoulife(noSerie: string): Promise<string> {
  try {
    const url = import.meta.env.VITE_API_URL_BASE + 'Vms/GetDahuaToken/' + noSerie;
    const tokenBear = localStorage.getItem('token_bear');

    const response = await fetch(url, {
      method: 'GET',
      headers: {
        'accept': 'application/json',
        'Content-Type': 'application/json-patch+json',
        'Authorization': 'Bearer ' + tokenBear,
      },
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    /**
     * EJEMPLO DE RESPUESTA
     * {
     *   "tokens": [
     *     {
     *       "channel": "7",     // Numero de canales del dispositivo
     *       "token": "Kt_or..." // TOKEN ImouLife
     *     }
     *   ]
     * }
     *
    */

    const data = await response.json();

    let token = '';
    if (data.tokens && data.tokens.length > 0) {
      token = data.tokens[0].token;
    }

    localStorage.setItem('token_imoulife', token);
    return token;
  } catch (error: any) {
    throw new Error(`Network error: ${error.message}`);
  }
}

const init = async () => {
  if (player) {
    destroy();
  }

  const { dvrPass, noSerie } = getURLparams();
  console.info("🚀 init player with params:", { dvrPass, noSerie });

  // localStorage.token_bear
  if (!localStorage.getItem('token_bear')) {
    try {
      await generateTokenBear();
    } catch (error) {
      console.error("❌ Error generating token Bearer:", error);
      hasError.value = true; // Actualiza el estado de error si falla la generación del token
      return;
    }
  }

  const token = await generateTokenimoulife(noSerie) ?? 'Kt_or2eaa394100304e8d9f301c52a66720';

  if (!dvrPass || !noSerie) {
    console.error("❌ Missing token or noSerie in URL params");
    hasError.value = true; // Actualiza el estado de error si faltan parámetros
    return;
  }

  hasError.value = false; // Resetea el estado de error si los parámetros son válidos

  player = new imouPlayer({
    id: "imou-player",
    width: 1200,
    height: 700,
    domain: "https://openapi-or.easy4ip.com",
    deviceId: noSerie, // Use noSerie as deviceId
    channelId: "7",
    token: token,
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
    code: noSerie,
    handleError: (err: unknown) => {
      console.error("handleError", err);
      hasError.value = true; // Actualiza el estado de error si ocurre un error
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
        token: localStorage.getItem('token_imoulife'),
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
  init();
});
</script>
<template>
  <div class="imou-player" :class="{ 'error-info': hasError }">
    <div
      id="imou-player"
      style="width: 1200px; height: 700px; background-color: #000"
    ></div>
    <div>
      <button style="display: none;" @click="init">Init</button>
      <button style="display: none;" @click="play">Play</button>
      <button style="display: none;" @click="pause">Pause</button>
      <button style="display: none;" @click="stop">Stop</button>
      <button style="display: none;" @click="capture">Capture</button>
      <button style="display: none;" @click="startTalk">Start Talk</button>
      <button style="display: none;" @click="stopTalk">Stop Talk</button>
      <button style="display: none;" @click="() => volume(1)">Open Volume</button>
      <button style="display: none;" @click="() => volume(0)">Close Volume</button>
      <button style="display: none;" @click="fullScreen">FullScreen</button>
      <button style="display: none;" @click="exitFullScreen">Exit FullScreen</button>
      <button style="display: none;" @click="startRecord">Start Screen Recording</button>
      <button style="display: none;" @click="stopRecord">Stop Screen Recording</button>
      <button style="display: none;" @click="initGrid">Init Grid x4 (same channel)</button>
    </div>
    <!-- Simple 2x2 grid containers -->
    <div class="player-grid" style="display: none;">
      <div v-for="id in gridIds" :key="id" :id="id" class="player-cell"></div>
    </div>
  </div>
</template>
<style scoped>
.error-info {
  /* Estilos para el estado de error */
  background-color: #f8d7da;
  color: #721c24;
  padding: 10px;
  border: 1px solid #f5c6cb;
}
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