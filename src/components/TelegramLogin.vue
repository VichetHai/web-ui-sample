<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const BOT_USERNAME = import.meta.env.VITE_TELEGRAM_BOT_USERNAME
const API_BASE = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000'

const user = ref(null)
const rawPayload = ref(null)
const apiResponse = ref(null)
const loading = ref(false)
const error = ref(null)

onMounted(() => {
  window.onTelegramAuth = handleTelegramAuth

  const script = document.createElement('script')
  script.async = true
  script.src = 'https://telegram.org/js/telegram-widget.js?22'
  // data-telegram-login requires the username without the leading '@'
  const botUsername = (BOT_USERNAME || '@YourBotUsername').replace(/^@/, '')
  script.setAttribute('data-telegram-login', botUsername)
  script.setAttribute('data-size', 'large')
  script.setAttribute('data-onauth', 'onTelegramAuth(user)')
  script.setAttribute('data-request-access', 'write')
  document.getElementById('telegram-btn').appendChild(script)
})

onUnmounted(() => {
  delete window.onTelegramAuth
})

async function handleTelegramAuth(data) {
  user.value = data
  rawPayload.value = data
  error.value = null
  apiResponse.value = null
  await postToApi(data)
}

async function postToApi(data) {
  loading.value = true
  try {
    const res = await fetch(`${API_BASE}/api/v1/profiles/register/telegram`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    })
    const text = await res.text()
    apiResponse.value = { status: res.status, body: text }
  } catch (err) {
    error.value = `API call failed: ${err.message}`
  } finally {
    loading.value = false
  }
}

function signOut() {
  user.value = null
  rawPayload.value = null
  apiResponse.value = null
  error.value = null
}
</script>

<template>
  <template v-if="!user">
    <p class="hint">Sign in with your Telegram account to test the OAuth flow.</p>
    <div id="telegram-btn" class="tg-btn-wrap"></div>
  </template>

  <template v-else>
    <div class="profile">
      <img v-if="user.photo_url" :src="user.photo_url" :alt="user.first_name" class="avatar" />
      <div v-else class="avatar-placeholder">{{ user.first_name?.[0] }}</div>
      <div>
        <p class="name">{{ user.first_name }} {{ user.last_name }}</p>
        <p class="email">@{{ user.username }}</p>
      </div>
    </div>

    <section class="section">
      <h2>Telegram Payload</h2>
      <pre>{{ JSON.stringify(rawPayload, null, 2) }}</pre>
    </section>

    <section class="section">
      <h2>API Response <code>POST /api/v1/profiles/register/telegram</code></h2>
      <div v-if="loading" class="loading">Sending to API…</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <pre v-else-if="apiResponse">Status: {{ apiResponse.status }}
{{ apiResponse.body }}</pre>
    </section>

    <button class="sign-out" @click="signOut">Sign out</button>
  </template>
</template>

<style scoped>
.tg-btn-wrap {
  display: flex;
  justify-content: center;
}

.avatar-placeholder {
  width: 56px;
  height: 56px;
  border-radius: 50%;
  background: #0088cc;
  color: #fff;
  font-size: 1.5rem;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
</style>
