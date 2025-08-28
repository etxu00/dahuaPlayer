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
const stop              = () => { 
  player.stop();
  hasError.value = false;
  window.close();
};
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
    channel: params.channel || "7",
  };
};

/**
 * Función auxiliar para obtener y guardar tokens en localStorage.
 */
function getStoredTokens(key: string): any[] {
  return localStorage.getItem(key) ? JSON.parse(localStorage.getItem(key) || '[]') : [];
}

function saveStoredTokens(key: string, tokens: any[]): void {
  localStorage.setItem(key, JSON.stringify(tokens));
}

/**
 * Verifica si un token es válido o necesita ser actualizado.
 */
function isTokenValid(expirationDate: string): boolean {
  return new Date(expirationDate) > new Date();
}

/**
 * Genera un token Bearer y lo guarda en localStorage.
 */
async function generateTokenBear(): Promise<string> {
  const username = import.meta.env.VITE_API_USERNAME;
  const password = import.meta.env.VITE_API_PASSWORD;
  const url = import.meta.env.VITE_API_URL_TOKEN_BEAR;

  const response = await fetch(url, {
    method: 'POST',
    headers: {
      'accept': 'application/json',
      'Content-Type': 'application/json-patch+json',
    },
    body: JSON.stringify({ username, password, rememberMe: true }),
  });

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`Error ${response.status}: ${errorText}`);
  }

  const data = await response.json();
  if (!data?.accessToken) {
    throw new Error('Invalid response from server');
  }

  const token = data.accessToken.token;
  const expirationDate = new Date(Date.now() + data.accessToken.expiresIn * 1000).toISOString();

  localStorage.setItem('token_bear', token);
  localStorage.setItem('token_bear_expiration', expirationDate);

  return token;
}

/**
 * Verifica o genera un nuevo token para un dispositivo (noSerie).
 */
async function getOrGenerateToken(noSerie: string): Promise<string> {
  const tokensImouLife = getStoredTokens('tokens_imoulife');
  const existingToken = tokensImouLife.find((token: any) => token.no_serie === noSerie);

  if (existingToken && isTokenValid(existingToken.expiration_date)) {
    console.info("✅ El token existente sigue siendo válido, reutilizándose.");
    return existingToken.token;
  }
  console.info("⚠️ El token falta o ha expirado, generando uno nuevo.");

  const tokenBear = localStorage.getItem('token_bear') || await generateTokenBear();
  const url = `${import.meta.env.VITE_API_URL_BASE}Vms/GetDahuaToken/${noSerie}`;

  const response = await fetch(url, {
    method: 'GET',
    headers: {
      'accept': 'application/json',
      'Content-Type': 'application/json-patch+json',
      'Authorization': `Bearer ${tokenBear}`,
    },
  });

  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }

  const data = await response.json();
  const newToken = {
    channels: data.tokens[0].channel,
    expiration_date: new Date(Date.now() + (import.meta.env.VITE_TOKEN_EXPIRATION_MINUTES || 7200) * 60 * 1000).toISOString(),
    no_serie: noSerie,
    token: data.tokens[0].token,
  };

  if (existingToken) {
    Object.assign(existingToken, newToken); // Actualiza el token existente
  } else {
    tokensImouLife.push(newToken); // Agrega un nuevo token
  }

  saveStoredTokens('tokens_imoulife', tokensImouLife);
  return newToken.token;
}

const init = async () => {
  if (player) {
    destroy();
  }

  const { dvrPass, noSerie, channel } = getURLparams();
  let tokensImouLife = null;
  console.info("🚀 iniciar reproductor con parámetros:", { dvrPass, noSerie });

  try {
    tokensImouLife = await getOrGenerateToken(noSerie);
    console.info("✅ Token generado con éxito:", tokensImouLife);
  } catch (error) {
    console.error("❌ Error al generar el token:", error);
    hasError.value = true; // Actualiza el estado de error si falla la generación del token
  }

  hasError.value = false; // Resetea el estado de error si los parámetros son válidos

  player = new imouPlayer({
    id: "imou-player",
    width: 1200,
    height: 700,
    domain: "https://openapi-or.easy4ip.com",
    deviceId: noSerie, // Use noSerie as deviceId
    channelId: channel || 7,
    token: tokensImouLife,
    type: 1, // 1 = Live; 2 = Playback
    streamId: 0, // Live 0-HD 高清; 1-SD 标清
    // 录播 云录像 cloud 本地录像 localRecord 默认 云录像
    // Playback, cloud-Cloud Video; localRecord-Local Video. Default Cloud Video
    recordType: "localRecord",     // or "localRecord"
    // beginTime: "2025-08-07 10:00:00",
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
    <div id="imou-player" style="width: 1200px; height: 700px; background-color: #000"></div>
    <div>
      <button style="display: none;" @click="init">Init</button>
      <button style="display: none;" @click="play">Play</button>
      <button style="display: none;" @click="pause">Pause</button>
      <button style="" @click="stop">Stop</button>
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