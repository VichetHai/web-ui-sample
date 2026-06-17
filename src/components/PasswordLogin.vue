<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const API_BASE = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000'
const CAPTCHA_ENABLED = import.meta.env.VITE_CAPTCHA_ENABLED === 'true'
const CAPTCHA_SITE_KEY = import.meta.env.VITE_CAPTCHA_SITE_KEY

const username = ref('')
const password = ref('')
const keyInfo = ref(null)
const keyLoading = ref(true)
const keyError = ref(null)

const captchaToken = ref(null)
const captchaWidgetId = ref(null)

const loading = ref(false)
const error = ref(null)
const apiResponse = ref(null)
const loggedIn = ref(false)

onMounted(() => {
  fetchPublicKey()
  if (CAPTCHA_ENABLED) loadTurnstile()
})

onUnmounted(() => {
  delete window.onTurnstileLoad
})

function loadTurnstile() {
  window.onTurnstileLoad = renderTurnstile
  if (window.turnstile) {
    renderTurnstile()
    return
  }
  const script = document.createElement('script')
  script.async = true
  script.defer = true
  script.src = 'https://challenges.cloudflare.com/turnstile/v0/api.js?onload=onTurnstileLoad'
  document.head.appendChild(script)
}

function renderTurnstile() {
  captchaWidgetId.value = window.turnstile.render('#turnstile-widget', {
    sitekey: CAPTCHA_SITE_KEY,
    callback: (token) => {
      captchaToken.value = token
    },
    'expired-callback': () => {
      captchaToken.value = null
    },
    'error-callback': () => {
      captchaToken.value = null
    },
  })
}

function resetTurnstile() {
  captchaToken.value = null
  if (window.turnstile && captchaWidgetId.value) {
    window.turnstile.reset(captchaWidgetId.value)
  }
}

async function fetchPublicKey() {
  keyLoading.value = true
  keyError.value = null
  try {
    const res = await fetch(`${API_BASE}/api/v1/auth/public-key`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    keyInfo.value = await res.json()
  } catch (err) {
    keyError.value = `Failed to load public key: ${err.message}`
  } finally {
    keyLoading.value = false
  }
}

function pemToArrayBuffer(pem) {
  const base64 = pem
    .replace(/-----BEGIN PUBLIC KEY-----/, '')
    .replace(/-----END PUBLIC KEY-----/, '')
    .replace(/\s+/g, '')
  const binary = atob(base64)
  const bytes = new Uint8Array(binary.length)
  for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i)
  return bytes.buffer
}

function bufferToHex(buffer) {
  return [...new Uint8Array(buffer)].map((b) => b.toString(16).padStart(2, '0')).join('')
}

async function encryptPassword(plain) {
  const { public_key, hash } = keyInfo.value
  const cryptoKey = await window.crypto.subtle.importKey(
    'spki',
    pemToArrayBuffer(public_key),
    { name: 'RSA-OAEP', hash: hash || 'SHA-256' },
    false,
    ['encrypt']
  )
  const encoded = new TextEncoder().encode(plain)
  const ciphertext = await window.crypto.subtle.encrypt({ name: 'RSA-OAEP' }, cryptoKey, encoded)
  return bufferToHex(ciphertext)
}

async function handleSubmit() {
  if (CAPTCHA_ENABLED && !captchaToken.value) {
    error.value = 'Please complete the captcha.'
    return
  }

  error.value = null
  apiResponse.value = null
  loading.value = true
  try {
    const encryptedPassword = keyInfo.value?.encryption_enabled
      ? await encryptPassword(password.value)
      : password.value

    const body = { username: username.value, password: encryptedPassword }
    if (CAPTCHA_ENABLED) body.captcha_token = captchaToken.value

    const res = await fetch(`${API_BASE}/api/v1/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    })
    const text = await res.text()
    apiResponse.value = { status: res.status, body: text }
    loggedIn.value = res.ok
  } catch (err) {
    error.value = `Login failed: ${err.message}`
  } finally {
    loading.value = false
    if (CAPTCHA_ENABLED) resetTurnstile()
  }
}

function signOut() {
  loggedIn.value = false
  apiResponse.value = null
  error.value = null
  username.value = ''
  password.value = ''
}
</script>

<template>
  <template v-if="!loggedIn">
    <p class="hint">Sign in with your username and password.</p>

    <div v-if="keyLoading" class="loading">Loading encryption key…</div>
    <div v-else-if="keyError" class="error">{{ keyError }}</div>

    <form v-else class="form" @submit.prevent="handleSubmit">
      <label class="field">
        <span>Username</span>
        <input v-model="username" type="text" required autocomplete="username" />
      </label>
      <label class="field">
        <span>Password</span>
        <input v-model="password" type="password" required autocomplete="current-password" />
      </label>
      <div v-if="CAPTCHA_ENABLED" id="turnstile-widget" class="captcha"></div>
      <button class="submit" type="submit" :disabled="loading || (CAPTCHA_ENABLED && !captchaToken)">
        {{ loading ? 'Signing in…' : 'Sign in' }}
      </button>
    </form>

    <div v-if="error" class="error">{{ error }}</div>

    <section v-if="apiResponse" class="section">
      <h2>API Response <code>POST /api/v1/auth/login</code></h2>
      <pre>Status: {{ apiResponse.status }}
{{ apiResponse.body }}</pre>
    </section>
  </template>

  <template v-else>
    <p class="hint">Logged in as <strong>{{ username }}</strong>.</p>

    <section class="section">
      <h2>API Response <code>POST /api/v1/auth/login</code></h2>
      <pre>Status: {{ apiResponse.status }}
{{ apiResponse.body }}</pre>
    </section>

    <button class="sign-out" @click="signOut">Sign out</button>
  </template>
</template>

<style scoped>
.form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  margin-bottom: 1rem;
}

.field {
  display: flex;
  flex-direction: column;
  gap: .35rem;
  font-size: .85rem;
  color: #444;
  font-weight: 500;
}

.field input {
  padding: .6rem .75rem;
  border: 1px solid #d0d0d0;
  border-radius: 6px;
  font-size: .95rem;
}

.field input:focus {
  outline: none;
  border-color: #888;
}

.submit {
  padding: .6rem 1.25rem;
  background: #1a1a1a;
  color: #fff;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: .95rem;
  font-weight: 500;
  transition: background .15s;
}

.submit:hover:not(:disabled) { background: #333; }
.submit:disabled { background: #999; cursor: not-allowed; }

.captcha {
  display: flex;
  justify-content: center;
}
</style>
