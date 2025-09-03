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
  await validateTokenImou() ;

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

// --------------------------------------------------

/**
 * VALIDACIÓN DEL TOKEN IMOU.   
 *
 * Verifica si existe un token válido para el dispositivo identificado por su número de serie.    
 * Si el token no existe o ha expirado, intenta obtener uno nuevo.
 *
 * @returns {string} `token` El token válido si existe y no ha expirado.
 * @returns {void} Si el token no existe, ha expirado y no se pudo obtener uno nuevo.
 *
 * @throws {Error} Si ocurre un error durante el proceso de validación o renovación del token.
 */
async function validateTokenImou(): Promise<string | void> {
  try {
    const tokensImou = getStoredTokensImou();
    const deviceToken = findTokenBySerie(tokensImou, _noSerie);

    if (!deviceToken) {
      await handleMissingToken(tokensImou);
      return;
    }

    if (isTokenExpired(deviceToken.expiration_date)) {
      await handleExpiredToken(tokensImou, deviceToken);
      return;
    }

    return handleValidToken(deviceToken.tokens, _channel);
  } catch (error) {
    handleError("Error al procesar el token del dispositivo.");
  }
}

/**
 * MANEJO DE TOKEN FALTANTE.   
 * 
 * Manejo de casos específicos en la validación del token.
 * @param {any} tokensImou - Lista de tokens almacenados.
 */
async function handleMissingToken(tokensImou: any): Promise<void> {
  await handleNewToken(tokensImou);
}

async function handleExpiredToken(tokensImou: any, tokenTMP: any): Promise<void> {
  await handleTokenRenewal(tokensImou, tokenTMP);
}

function handleValidToken(tokens: any, channel: any): string {
  return getChannelToken(tokens, channel);
}

/**
 * OBTENCIÓN DE TOKEN IMOU ALMACENADOS.    
 * 
 * Obtiene los tokens almacenados en el `localStorage`.
 * @returns {any[]} Lista de tokens almacenados.
 */
function getStoredTokensImou(): any[] {
  return JSON.parse(localStorage.getItem('tokens_imou') || '[]');
}

/**
 * BÚSQUEDA DE TOKEN POR NÚMERO DE SERIE.   
 * 
 * Busca los tokens de un dispositivo específico por su número de serie. El item contiene    
 * los tokens para cada canal del dispositivo (DRV).
 * @param {any[]} tokens - Lista de tokens almacenados.
 * @param {string} noSerie - Número de serie del dispositivo (DRV).
 * @returns {any} El token encontrado o `undefined` si no se encuentra.
 */
function findTokenBySerie(tokens: any[], noSerie: string): any {
  return tokens.find((token: any) => token.no_serie === noSerie);
}

/**
 * VERIFICACIÓN DE EXPIRACIÓN DEL TOKEN.    
 * 
 * Verifica si un token ha expirado comparando la fecha actual con la fecha de expiración del token.
 * @param {string} expirationDate - Fecha de expiración del token en formato ISO.
 * @returns {boolean} `true` si el token ha expirado, `false` en caso contrario.
 */
function isTokenExpired(expirationDate: string): boolean {
  return new Date() >= new Date(expirationDate);
}

/**
 * MANEJO DE TOKEN NUEVO Y RENOVACIÓN.    
 * 
 * Funciones para manejar la obtención y renovación de tokens Imou.
 * 
 * @param {any} tokensImou - Lista de tokens almacenados.
 */
async function handleNewToken(tokensImou: any[]): Promise<void> {
  try {
    const newToken = await getTokenImou();
    const expirationDate = calculateExpirationDate();

    tokensImou.push({
      expiration_date: expirationDate,
      issue_date: new Date().toISOString(),
      no_serie: _noSerie,
      tokens: newToken.tokens,
    });

    _tokenImou = getChannelToken(newToken.tokens, _channel) || "";
    saveTokens(tokensImou);
  } catch {
    handleError("Error al obtener el token del dispositivo.");
  }
}

async function handleTokenRenewal(tokensImou: any[], tokenTMP: any): Promise<void> {
  try {
    const newToken = await getTokenImou();
    tokenTMP.tokens = newToken.tokens;
    tokenTMP.expiration_date = calculateExpirationDate();

    saveTokens(tokensImou);
  } catch {
    handleError("Error al renovar el token del dispositivo.");
  }
}

/**
 * OBTENER TOKEN DE CANAL ESPECÍFICO.
 */
function getChannelToken(tokens: any[], channel: string): string | void {
  const channelIndex = Number(channel) - 1;
  const channelToken = tokens.find((t: any) => t.channel.toString() === channelIndex.toString());

  if (channelToken) {
    loading.value = false;
    return channelToken.token;
  } else {
    handleError("El canal especificado no existe en el dispositivo.");
  }
}

function calculateExpirationDate(): string {
  const expirationMinutes = (import.meta.env.VITE_TOKEN_EXPIRATION_MINUTES_HRS || 7200) * 60 * 1000;
  return new Date(Date.now() + expirationMinutes).toISOString();
}

function saveTokens(tokens: any[]): void {
  localStorage.setItem('tokens_imou', JSON.stringify(tokens));
}

function handleError(message: string): void {
  loading.value = false;
  hasError.value = true;
  msgError.value = message;
}

async function _validateTokenImou(): Promise<any> {
  const tokensImou = localStorage.getItem('tokens_imou')
    ? JSON.parse(localStorage.getItem('tokens_imou') || '[]')
    : [];

  const tokenTMP = tokensImou.find((token: any) => token.no_serie === _noSerie);

  if (!tokenTMP) {
    // Obtenemos un nuevo token
    try {
      const newToken = await getTokenImou();

      tokensImou.push({
        expiration_date: new Date(Date.now() + (import.meta.env.VITE_TOKEN_EXPIRATION_MINUTES || 7200)).toISOString(),
        issue_date: new Date().toISOString(),
        no_serie: _noSerie,
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

  return token;
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
  const params: Record<string, string> = {};
  const urlParams = new URLSearchParams(window.location.search);
  urlParams.forEach((value, key) => params[key] = value);
  _channel = params.channel || "";
  _password = decryptPassword(params.token) || "";
  _noSerie = params.noSerie || "";
};

function decryptPassword(encryptedPassword: string): string {
  const secretKey = import.meta.env.VITE_ENCRYPTION_KEY || '';
  try {
    if (!encryptedPassword || !secretKey) {
      throw new Error("La contraseña o la clave de descifrado no están definidas.");
    }
    const bytes = CryptoJS.AES.decrypt(encryptedPassword, secretKey);
    const decryptedPassword = bytes.toString(CryptoJS.enc.Utf8);

    if (!decryptedPassword) {
      throw new Error("La contraseña descifrada está vacía.");
    }

    return decryptedPassword;
  } catch (error) {
    console.error("Error al descifrar la contraseña:", error);
    throw error; // Lanza el error para manejarlo en otro lugar si es necesario
  }
}

/**
 * onMounted se ejecuta cuando el componente se monta en el DOM.
 */
onMounted(() => inicio());

const cargando = ref(true); // Variable reactiva para manejar el estado de error
const errorGeneral = ref(false); // Variable reactiva para manejar el estado de error
const descripcionError = ref(""); // Variable para almacenar el mensaje de error

let _canal = "1";
let _contrasena = "";
let _numSerie = "";
let _tokensDRV:any = null;
let _tokenCanal:any = null;
let _tokenBearer:any = null;

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

async function iniciarStream() {
  if (errorGeneral.value) {
    return
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
      hasError.value = true; // Actualiza el estado de error si ocurre un error
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
    await iniciarStream();
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