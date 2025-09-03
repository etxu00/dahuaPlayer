<script lang="ts" setup>
import { ref, onMounted } from "vue";
import CryptoJS from 'crypto-js';

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

const play              = () => { player.play() };
const pause             = () => { player.pause() };
const stop              = () => { player.stop(); errorGeneral.value = false; setTimeout(() => window.close(), 1000)};
const capture           = () => { player.capture() };
const startTalk         = () => { player.startTalk() };
const stopTalk          = () => { player.stopTalk() };
const volume            = (value: number) => { player.volume(value) };
const fullScreen        = () => { player.fullScreen() };
const exitFullScreen    = () => { player.exitFullScreen() };
const startRecord       = () => { player.startRecord() };
const stopRecord        = () => { player.stopRecord() };
const destroy           = () => { player.destroy(); player = null!};

const destroyGrid = () => {
  gridPlayers.forEach(p => { try { p && p.destroy && p.destroy(); } catch {} });
  gridPlayers = [];
};

const initGrid = () => {
  destroyGrid();

  const tokens = localStorage.getItem('tokens_imou')
    ? JSON.parse(localStorage.getItem('tokens_imou') || '[]')
    : [];
  
  const tokenTMP = tokens.find((token: any) => token.no_serie === _numSerie);
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
        deviceId: _numSerie,
        channelId: gridChannels[index] ? Number(gridChannels[index].channel) : 0,
        token: gridChannels[index] ? gridChannels[index].token : _tokensDRV,
        type: 1, // Live
        streamId: 1, // Use SD (1) instead of HD (0) for better performance with multiple streams
        recordType: "cloud", 
        muted: true,
        code: _contrasena,
        handleError: (err: unknown) => console.error(id, err),
      });
      gridPlayers.push(p);
    }, index * 500); // 500ms delay between each player
  });
};
const cargando = ref(true); // Variable reactiva para manejar el estado de error
const errorGeneral = ref(false); // Variable reactiva para manejar el estado de error
const descripcionError = ref(""); // Variable para almacenar el mensaje de error

let _canal = "1";
let _contrasena = "";
let _numSerie = "";
let _tokensDRV:any = null;
let _tokenCanal:any = null;
let _tokenBearer:any = null;

let player: IPlayer;
let gridIds = ["cell-0", "cell-1"];
let gridPlayers: any[] = [];

function desencriptar(contrasena: string): string {
  const secretKey = import.meta.env.VITE_ENCRYPTION_KEY || '';
  try {
    if (!contrasena || !secretKey) {
      throw new Error("La contraseña o la clave de descifrado no están definidas.");
    }

    const bytes = CryptoJS.AES.decrypt(contrasena, secretKey);
    const contrasenaFinal = bytes.toString(CryptoJS.enc.Utf8);

    if (!contrasenaFinal) {
      throw new Error("La contraseña descifrada está vacía.");
    }

    return contrasenaFinal;
  } catch (error) {
    console.error("Error al descifrar la contraseña:", error);
    throw error; // Lanza el error para manejarlo en otro lugar si es necesario
  }
}

async function generarTokenBearer() {
  const data: any = await obtenerTokenBearer();
  const tokenBear = data?.accessToken?.token || ""
  const fechaExpiracion = data?.accessToken?.expiresIn || 0
  const tokenBearExpiracion = new Date(Date.now() + fechaExpiracion * 1000).toISOString()

  localStorage.setItem('token_bear', tokenBear)
  localStorage.setItem('token_bear_expiration', tokenBearExpiracion)

  _tokenBearer = tokenBear
}

async function generarTokenDRV() {
  await validarTokenBearer()
  await validarVigenciaTokenBearer()

  const data: any = await obtenerTokenDVR()
  const expirationMinutes = import.meta.env.VITE_TOKEN_EXPIRATION_MINUTES || 7200
  const tokensImou = JSON.parse(localStorage.getItem('tokens_imou') || '[]')
  const tokenDRV = tokensImou.find((token: any) => token.no_serie === _numSerie)
  const nuevaFechaExpiracion = new Date(Date.now() + expirationMinutes * 60 * 1000).toISOString()

  if (tokenDRV) {
    tokenDRV.expiration_date = nuevaFechaExpiracion
    tokenDRV.issue_date = new Date().toISOString()
    tokenDRV.tokens = data.tokens
  } else {
    tokensImou.push({
      expiration_date: nuevaFechaExpiracion,
      issue_date: new Date().toISOString(),
      no_serie: _numSerie,
      tokens: data.tokens,
    })
  }

  localStorage.setItem('tokens_imou', JSON.stringify(tokensImou))
  _tokensDRV = tokensImou.find((token: any) => token.no_serie === _numSerie)
  _tokenCanal = _tokensDRV.tokens.find((t: any) => t.channel.toString() === (Number(_canal) - 1).toString())
}

function iniciarStream() {
  if (errorGeneral.value) {
    return
  }

  const playerContainer = document.getElementById("imou-player");
  if (!playerContainer) {
    console.error("El contenedor 'imou-player' no existe en el DOM.");
    errorGeneral.value = true;
    cargando.value = false;
    descripcionError.value = "Error interno: contenedor de reproductor no encontrado."
    return;
  }

  const deviceId = _numSerie // Use noSerie as deviceId
  const channelId = Number(_canal) - 1 // Se requiere restar 1. Índice basado en cero (Canal 1 = 0, Canal 2 = 1, etc.)
  const token = _tokenCanal.token
  const code = _contrasena
  player = new imouPlayer({
    id            : "imou-player",
    width         : 1200,
    height        : 700,
    domain        : "https://openapi-or.easy4ip.com",
    deviceId      : deviceId,
    channelId     : channelId,
    token         : token,
    type          : 1, // 1 = Live; 2 = Playback
    streamId      : 0, // Live 0-HD 高清; 1-SD 标清
    // 录播 云录像 cloud 本地录像 localRecord 默认 云录像
    // Playback, cloud-Cloud Video; localRecord-Local Video. Default Cloud Video
    recordType    : "localRecord",     // or "localRecord"
    // beginTime: "2025-08-07 10:00:00",
    // endTime:   "2025-08-07 10:30:00",
    muted         : false,
    code          : code,
    handleError   : (err: unknown) => {
      console.error("handleError", err);
      errorGeneral.value = true; // Actualiza el estado de error si ocurre un error
    },
  })
  window.player = player
}

async function inicio() {
  try {
    await obtenerParametrosURL()
    await validarParametrosURL();
    await validarTokenDRV();
    await validarVigenciaTokenDRV();
    iniciarStream();
  } catch (error) {
    console.error("Error en el flujo de inicio:", error)
    cargando.value = false
    errorGeneral.value = true
    descripcionError.value = "Ocurrió un error durante la inicialización."
  }
}

async function obtenerParametrosURL() {
  const params: Record<string, string> = {}
  const urlParams = new URLSearchParams(window.location.search)
  urlParams.forEach((value, key) => params[key] = value)
  _canal = params.channel || ""
  _contrasena = params.token || ""
  _numSerie = params.noSerie || ""
}

async function obtenerTokenBearer(): Promise<void> {
  try {
    const username = import.meta.env.VITE_API_USERNAME
    const password = import.meta.env.VITE_API_PASSWORD
    const url = import.meta.env.VITE_API_URL_TOKEN_BEAR
    const response = await fetch(url, {
      method: 'POST',
      headers: { 'accept': 'application/json', 'Content-Type': 'application/json-patch+json' },
      body: JSON.stringify({ username, password, rememberMe: true }),
    })

    if (!response.ok) {
      errorGeneral.value = true
      descripcionError.value = `Error en la solicitud: ${response.status} ${response.statusText}`
      throw new Error(`Error en la solicitud: ${response.status} ${response.statusText}`);
    }

    return await response.json()
  } catch (error) {
    console.error(error)
    errorGeneral.value = true
    descripcionError.value = "Error en la conexión con la autenticación."
    throw error
  }
}

async function obtenerTokenDVR(): Promise<void> {
  try {
    const url = `${import.meta.env.VITE_API_URL_BASE}Vms/GetDahuaToken/${_numSerie}`
    const response = await fetch(url, {
      method: 'GET',
      headers: {
        'accept': 'application/json',
        'Content-Type': 'application/json-patch+json',
        'Authorization': `Bearer ${_tokenBearer}`,
      },
    })

    if (!response.ok) {
      errorGeneral.value = true
      descripcionError.value = `Error en la solicitud: ${response.status} ${response.statusText}`
      throw new Error(`Error en la solicitud: ${response.status} ${response.statusText}`);
    }

    return await response.json()
  } catch (error) {
    console.error(error)
    errorGeneral.value = true
    descripcionError.value = "Error en la conexión con el dispositivo."
    throw error
  }
}

async function validarParametrosURL() {
  if (!_numSerie || !_contrasena || !_canal || isNaN(Number(_canal))) {
    cargando.value = false
    errorGeneral.value = true
    descripcionError.value = "Información del dispositivo incorrecta o faltante."
    return;
  } else {
    _contrasena = desencriptar(_contrasena)
  }
}

async function validarTokenBearer() {
  const tokenBearer = localStorage.getItem('token_bear')
  if (tokenBearer) {
    _tokenBearer = tokenBearer
  } else {
    await generarTokenBearer()
  }
}

async function validarTokenDRV() {
  const tokensImou = JSON.parse(localStorage.getItem('tokens_imou') || '[]')
  const tokenDVR = tokensImou.find((token: any) => token.no_serie === _numSerie)
  if (!tokenDVR) {
    await generarTokenDRV()
  } else {
    _tokensDRV = tokenDVR;
    _tokenCanal = _tokensDRV.tokens.find((t: any) => t.channel.toString() === (Number(_canal) - 1).toString())
  }
}

async function validarVigenciaTokenBearer() {
  const expiration = localStorage.getItem('token_bear_expiration')
  if (expiration) {
    const now = new Date()
    const expDate = new Date(expiration)
    if (now >= expDate) {
      await generarTokenBearer()
    }
  } else {
    await generarTokenBearer()
  }
}

async function validarVigenciaTokenDRV() {
  const fechaExpiracion = new Date(_tokensDRV.expiration_date)
  const fechaActual = new Date()
  if (fechaActual >= fechaExpiracion) {
    await generarTokenDRV()
  }
}

/**
 * onMounted se ejecuta cuando el componente se monta en el DOM.
 */
onMounted(() => {
  inicio()
  cargando.value = false
});
</script>
<template>
  <div v-if="cargando">Cargando...</div>
  <div class="error-info" v-if="errorGeneral"><p>{{ descripcionError }}</p></div>
  <div class="imou-player" v-if="!errorGeneral && !cargando">
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