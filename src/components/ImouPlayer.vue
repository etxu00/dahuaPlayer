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

function reset() {
  // Elimina toda la información de localStorage y recarga la página
  localStorage.removeItem('tokens_imou');
  localStorage.removeItem('token_bear');
  localStorage.removeItem('token_bear_expiration');
  window.location.reload();
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
 * onMounted se ejecuta cuando el componente se monta en el DOM.4u!'5eIs0n\P
 */
onMounted(() => {
  inicio()
  cargando.value = false
});
</script>
<template>
  <div v-if="cargando">Cargando...</div>
  <div class="error-info" v-if="errorGeneral"><p>{{ descripcionError }}</p></div>
  <header>
    <div class="container">
      <div class="btn">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
          fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
          stroke-linejoin="round">
          <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
          <path d="M12 12m-1 0a1 1 0 1 0 2 0a1 1 0 1 0 -2 0" />
          <path d="M12 19m-1 0a1 1 0 1 0 2 0a1 1 0 1 0 -2 0" />
          <path d="M12 5m-1 0a1 1 0 1 0 2 0a1 1 0 1 0 -2 0" />
        </svg>
      </div>
      <div class="dropdown">
        <button @click="play">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M7 4v16l13 -8z" />
          </svg>
          <span>Reproducir</span>
        </button>
        <button @click="pause">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M6 5m0 1a1 1 0 0 1 1 -1h2a1 1 0 0 1 1 1v12a1 1 0 0 1 -1 1h-2a1 1 0 0 1 -1 -1z" />
            <path d="M14 5m0 1a1 1 0 0 1 1 -1h2a1 1 0 0 1 1 1v12a1 1 0 0 1 -1 1h-2a1 1 0 0 1 -1 -1z" />
          </svg>
          <span>Pausa</span>
        </button>
        <button style="" @click="stop">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M5 5m0 2a2 2 0 0 1 2 -2h10a2 2 0 0 1 2 2v10a2 2 0 0 1 -2 2h-10a2 2 0 0 1 -2 -2z" />
          </svg>
          <span>Detener</span>
        </button>
        <button @click="capture">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M5 7h1a2 2 0 0 0 2 -2a1 1 0 0 1 1 -1h6a1 1 0 0 1 1 1a2 2 0 0 0 2 2h1a2 2 0 0 1 2 2v9a2 2 0 0 1 -2 2h-14a2 2 0 0 1 -2 -2v-9a2 2 0 0 1 2 -2" />
            <path d="M9 13a3 3 0 1 0 6 0a3 3 0 0 0 -6 0" />
          </svg>
          <span>Captura</span>
        </button>
        <button @click="startTalk">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M9 2m0 3a3 3 0 0 1 3 -3h0a3 3 0 0 1 3 3v5a3 3 0 0 1 -3 3h0a3 3 0 0 1 -3 -3z" />
            <path d="M5 10a7 7 0 0 0 14 0" />
            <path d="M8 21l8 0" />
            <path d="M12 17l0 4" />
          </svg>
          <span>Iniciar conversación</span>
        </button>
        <button @click="stopTalk">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M3 3l18 18" />
            <path d="M9 5a3 3 0 0 1 6 0v5a3 3 0 0 1 -.13 .874m-2 2a3 3 0 0 1 -3.87 -2.872v-1" />
            <path d="M5 10a7 7 0 0 0 10.846 5.85m2 -2a6.967 6.967 0 0 0 1.152 -3.85" />
            <path d="M8 21l8 0" />
            <path d="M12 17l0 4" />
          </svg>
          <span>Detener conversación</span>
        </button>
        <button @click="() => volume(1)">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M15 8a5 5 0 0 1 0 8" />
            <path d="M17.7 5a9 9 0 0 1 0 14" />
            <path d="M6 15h-2a1 1 0 0 1 -1 -1v-4a1 1 0 0 1 1 -1h2l3.5 -4.5a.8 .8 0 0 1 1.5 .5v14a.8 .8 0 0 1 -1.5 .5l-3.5 -4.5" />
          </svg>
          <span>Activar volumen</span>
        </button>
        <button @click="() => volume(0)">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M15 8a5 5 0 0 1 1.912 4.934m-1.377 2.602a5 5 0 0 1 -.535 .464" />
            <path d="M17.7 5a9 9 0 0 1 2.362 11.086m-1.676 2.299a9 9 0 0 1 -.686 .615" />
            <path d="M9.069 5.054l.431 -.554a.8 .8 0 0 1 1.5 .5v2m0 4v8a.8 .8 0 0 1 -1.5 .5l-3.5 -4.5h-2a1 1 0 0 1 -1 -1v-4a1 1 0 0 1 1 -1h2l1.294 -1.664" />
            <path d="M3 3l18 18" />
          </svg>
          <span>Desactivar volumen</span>
        </button>
        <button @click="fullScreen">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M16 4l4 0l0 4" />
            <path d="M14 10l6 -6" />
            <path d="M8 20l-4 0l0 -4" />
            <path d="M4 20l6 -6" />
            <path d="M16 20l4 0l0 -4" />
            <path d="M14 14l6 6" />
            <path d="M8 4l-4 0l0 4" />
            <path d="M4 4l6 6" />
          </svg>
          <span>Pantalla completa</span>
        </button>
        <button @click="exitFullScreen">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M5 9l4 0l0 -4" />
            <path d="M3 3l6 6" />
            <path d="M5 15l4 0l0 4" />
            <path d="M3 21l6 -6" />
            <path d="M19 9l-4 0l0 -4" />
            <path d="M15 9l6 -6" />
            <path d="M19 15l-4 0l0 4" />
            <path d="M15 15l6 6" />
            </svg>
          <span>Salir de pantalla completa</span>
        </button>
        <button @click="startRecord">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M8 5.072a8 8 0 1 1 -3.995 7.213l-.005 -.285l.005 -.285a8 8 0 0 1 3.995 -6.643z" />
            </svg>
          <span>Iniciar grabación de pantalla</span>
        </button>
        <button @click="stopRecord">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <svg  xmlns="http://www.w3.org/2000/svg"  width="24"  height="24"  viewBox="0 0 24 24"  fill="currentColor">
              <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
              <path d="M17 4h-10a3 3 0 0 0 -3 3v10a3 3 0 0 0 3 3h10a3 3 0 0 0 3 -3v-10a3 3 0 0 0 -3 -3z" />
            </svg>
          </svg>
          <span>Detener grabación de pantalla</span>
        </button>
        <button @click="initGrid">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M4 4m0 1a1 1 0 0 1 1 -1h4a1 1 0 0 1 1 1v4a1 1 0 0 1 -1 1h-4a1 1 0 0 1 -1 -1z" />
            <path d="M14 4m0 1a1 1 0 0 1 1 -1h4a1 1 0 0 1 1 1v4a1 1 0 0 1 -1 1h-4a1 1 0 0 1 -1 -1z" />
            <path d="M4 14m0 1a1 1 0 0 1 1 -1h4a1 1 0 0 1 1 1v4a1 1 0 0 1 -1 1h-4a1 1 0 0 1 -1 -1z" />
            <path d="M14 14m0 1a1 1 0 0 1 1 -1h4a1 1 0 0 1 1 1v4a1 1 0 0 1 -1 1h-4a1 1 0 0 1 -1 -1z" />
          </svg>
          <span>Iniciar cuadrícula</span>
        </button>
        <button @click="reset">
          <svg  xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"
            fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
            stroke-linejoin="round">
            <path stroke="none" d="M0 0h24v24H0z" fill="none"/>
            <path d="M3.06 13a9 9 0 1 0 .49 -4.087" />
            <path d="M3 4.001v5h5" />
            <path d="M12 12m-1 0a1 1 0 1 0 2 0a1 1 0 1 0 -2 0" />
          </svg>
          <span>Forzar reintento</span>
        </button>
      </div>
    </div>
  </header>
  <div class="imou-player" v-if="!errorGeneral && !cargando">
    <div id="imou-player" style="width: 1200px; height: 700px; background-color: #000">
    </div>
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
.kind-stream-canvas.video-player-canvas {
  left: 0;
}
</style>
<style scoped>
  .container {
    width: fit-content;
    margin-left: auto;
  }
  header {
    background-color: #333;
    display: flex;
    left: 0;
    padding: .5rem;
    position: fixed;
    top: 0;
    width: 100dvw;
    z-index: 2;

    & + * {
      margin-top: 3rem;
    }
  }
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

  .imou-player * {z-index: 0;}

  svg {
    width: 18px;
    height: 18px;
  }

  button,
  .btn {
    display: flex;
    align-items: center;
    gap: .5rem;
    width: 100%;
    background-color: transparent;
    border: none;
    color: inherit;
    margin: 0;
    padding: .25rem .5rem;
    text-align: left;

    &:hover {
      color: orange;
      cursor: pointer;
      background-color: rgba(255, 165, 0, 0.15);
    }
  }

  .dropdown:not(:hover) {
    display: none;
  }

  .container:hover .dropdown,
  .btn:hover + .dropdown {
    display: flex;
  }

  .dropdown {
    background-color: #333;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
    display: flex;
    flex-direction: column;
    gap: .25rem;
    padding: .5rem 0;
    border-radius: .25rem;
    width: 250px;
    position: absolute;
    top: calc(100% - .5rem);
    right: 0;
  }

  div.btn {
    border-radius: 50%;
    margin-left: auto;
    width: fit-content;
  }
</style>