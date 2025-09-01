<script lang="ts" setup>
import { ref, onMounted } from "vue";

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
let _channel = "1";
let _password = "";
let _noSerie = "";
let _tokenImou = "";
let gridIds = ["cell-0", "cell-1"];

const loading = ref(true); // Variable reactiva para manejar el estado de error
const hasError = ref(false); // Variable reactiva para manejar el estado de error
const msgError = ref(""); // Variable para almacenar el mensaje de error
const play              = () => { player.play() };
const pause             = () => { player.pause() };
const stop              = () => { 
  player.stop();
  hasError.value = false;
  setTimeout(() => window.close(), 1000);
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
let gridPlayers: any[] = [];

const destroyGrid = () => {
  gridPlayers.forEach(p => {
    try { p && p.destroy && p.destroy(); } catch {}
  });
  gridPlayers = [];
};

const initGrid = () => {
  destroyGrid();

  const tokens = localStorage.getItem('tokens_imou')
    ? JSON.parse(localStorage.getItem('tokens_imou') || '[]')
    : [];
  
  const tokenTMP = tokens.find((token: any) => token.no_serie === _noSerie);
  const gridChannels = tokenTMP.tokens;

  if (!gridChannels.length) {
    return;
  }

  gridChannels.forEach((channel: any, index: number) => {
    gridIds[index] = `cell-${index}`;
  });

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
        deviceId: _noSerie,
        channelId: gridChannels[index] ? Number(gridChannels[index].channel) : 0,
        token: gridChannels[index] ? gridChannels[index].token : _tokenImou,
        type: 1, // Live
        streamId: 1, // Use SD (1) instead of HD (0) for better performance with multiple streams
        recordType: "cloud", 
        muted: true,
        code: _noSerie,
        handleError: (err: unknown) => console.error(id, err),
      });
      gridPlayers.push(p);
    }, index * 500); // 500ms delay between each player
  });
};

const init = async () => {
  // 1. Obtenemos los parametros de la URL
  valideteURLparams();

  // 2. Validamos si existe token para el Dispositivo
  _tokenImou = await validateTokenImou();

  hasError.value = false; // Resetea el estado de error si los parámetros son válidos

  player = new imouPlayer({
    id: "imou-player",
    width: 1200,
    height: 700,
    domain: "https://openapi-or.easy4ip.com",
    deviceId: _noSerie, // Use noSerie as deviceId
    channelId: Number(_channel) - 1,
    token: _tokenImou,
    type: 1, // 1 = Live; 2 = Playback
    streamId: 0, // Live 0-HD 高清; 1-SD 标清
    // 录播 云录像 cloud 本地录像 localRecord 默认 云录像
    // Playback, cloud-Cloud Video; localRecord-Local Video. Default Cloud Video
    recordType: "localRecord",     // or "localRecord"
    // beginTime: "2025-08-07 10:00:00",
    // endTime:   "2025-08-07 10:30:00",
    muted: false,
    code: _noSerie,
    handleError: (err: unknown) => {
      console.error("handleError", err);
      hasError.value = true; // Actualiza el estado de error si ocurre un error
    },
  });
  window.player = player;
}

/**
 * Verifica si ya existe un token y si no ha expirado.
 */
function validateTokenBearer(): string {
  const token = localStorage.getItem('token_bear');
  const expiration = localStorage.getItem('token_bear_expiration');

  if (token && expiration) {
    const now = new Date();
    const expDate = new Date(expiration);
    if (now < expDate) {
      return token; // Token válido
    }
  }
  return ""; // No hay token o ha expirado
}

function valideteURLparams() {
  getURLparams();
  if (!_noSerie || !_password || !_channel) {
    loading.value = false;
    hasError.value = true;
    msgError.value = "Error en la información del dispositivo. No se puede continuar.";
    return;
  }
}

function generateGrid(token: string) {
  _tokenImou = token;
  initGrid();
  loading.value = false;
}

async function validateTokenImou(): Promise<any> {
  const tokensImou = localStorage.getItem('tokens_imou')
    ? JSON.parse(localStorage.getItem('tokens_imou') || '[]')
    : [];

  const tokenTMP = tokensImou.find((token: any) => token.no_serie === _noSerie);

  if (!tokenTMP) {
    // Obtenemos un nuevo token
    try {
      const newToken = await getTokenImou();

      tokensImou.push({
        no_serie: _noSerie,
        expiration_date: new Date(Date.now() + (import.meta.env.VITE_TOKEN_EXPIRATION_MINUTES || 7200) * 60 * 1000).toISOString(),
        tokens: newToken.tokens,
      });

      localStorage.setItem('tokens_imou', JSON.stringify(tokensImou));
    } catch (error) {
      loading.value = false;
      hasError.value = true;
      msgError.value = "Error al obtener el token del dispositivo.";
    }
  } else {
    const now = new Date();
    const expDate = new Date(tokenTMP.expiration_date);
    
    if (now >= expDate) {
      // Token expirado, obtener uno nuevo
      try {
        const newToken = await getTokenImou();

        tokenTMP.tokens = newToken.tokens;
        tokenTMP.expiration_date = new Date(Date.now() + (import.meta.env.VITE_TOKEN_EXPIRATION_MINUTES || 7200) * 60 * 1000).toISOString();

        localStorage.setItem('tokens_imou', JSON.stringify(tokensImou));
      } catch (error) {
        loading.value = false;
        hasError.value = true;
        msgError.value = "Error al renovar el token del dispositivo.";
      }
    } else {
      
      // Buscamos el canal específico
      const channel = Number(_channel) - 1; // Ajuste para índice basado en cero
      const channelToken = tokenTMP.tokens.find((t: any) => t.channel.toString() === channel.toString());

      if (channelToken) {
        loading.value = false;
        return channelToken.token; // Retorna el token del canal específico
      } else {
        loading.value = false;
        hasError.value = true;
        msgError.value = "El canal especificado no existe en el dispositivo.";
      }


    }
  }
}

async function getTokenBearer(): Promise<string> {
  const username = import.meta.env.VITE_API_USERNAME;
  const password = import.meta.env.VITE_API_PASSWORD;
  const url = import.meta.env.VITE_API_URL_TOKEN_BEAR;
  const response = await fetch(url, {
    method: 'POST',
    headers: { 'accept': 'application/json', 'Content-Type': 'application/json-patch+json' },
    body: JSON.stringify({ username, password, rememberMe: true }),
  });

  const data = await response.json();
  const token = data.accessToken.token;
  const expirationDate = new Date(Date.now() + data.accessToken.expiresIn * 1000).toISOString();

  localStorage.setItem('token_bear', token);
  localStorage.setItem('token_bear_expiration', expirationDate);

  return data
}

async function getTokenImou(): Promise<any> {
  const url = `${import.meta.env.VITE_API_URL_BASE}Vms/GetDahuaToken/${_noSerie}`;
  const tokenBear = validateTokenBearer() || await getTokenBearer();
  const response = await fetch(url, {
    method: 'GET',
    headers: {
      'accept': 'application/json',
      'Content-Type': 'application/json-patch+json',
      'Authorization': `Bearer ${tokenBear}`,
    },
  });

  const data = await response.json();
  return data;
}

const getURLparams = () => {
  // Example .../index.html?token=...&noSerie=...&channel=1
  // Se utiliza token en la URL para no indicar que es una contraseña. // TODO: encriptar datos

  const params: Record<string, string> = {};
  const urlParams = new URLSearchParams(window.location.search);
  urlParams.forEach((value, key) => params[key] = value);
  _channel = params.channel || "";
  _password = params.token || "";
  _noSerie = params.noSerie || "";
};

/**
 * onMounted se ejecuta cuando el componente se monta en el DOM.
 */
onMounted(() => init());
</script>
<template>
  <div v-if="loading">
    Cargando...
  </div>
  <div class="error-info" v-if="hasError">
    <p>{{ msgError }}</p>
  </div>
  <div class="imou-player" v-if="!hasError && !loading">
    <div id="imou-player" style="width: 1200px; height: 700px; background-color: #000"></div>
    <div>
      <button @click="play">Reproducir</button>
      <button @click="pause">Pausa</button>
      <button style="" @click="stop">Detener</button>
      <button @click="capture">Captura</button>
      <button @click="startTalk">Iniciar conversación</button>
      <button @click="stopTalk">Detener conversación</button>
      <button @click="() => volume(1)">Activar volumen</button>
      <button @click="() => volume(0)">Desactivar volumen</button>
      <button @click="fullScreen">Pantalla completa</button>
      <button @click="exitFullScreen">Salir de pantalla completa</button>
      <button @click="startRecord">Iniciar grabación de pantalla</button>
      <button @click="stopRecord">Detener grabación de pantalla</button>
      <button @click="initGrid">Iniciar cuadrícula x4 (mismo canal)</button>
    </div>
    <!-- Simple 2x2 grid containers -->
    <div class="player-grid">
      <div v-for="id in gridIds" :key="id" :id="id" class="player-cell"></div>
    </div>
  </div>
</template>
<style>
*, *:before, *:after {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: Arial, sans-serif;
  background-color: black;
  color: white;
}
</style>
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