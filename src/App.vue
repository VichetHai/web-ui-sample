<script setup>
import { ref, onMounted } from 'vue'
import TelegramLogin from './components/TelegramLogin.vue'

const tab = ref('google')

const CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID

const user = ref(null)
const idToken = ref(null)
const apiResponse = ref(null)
const loading = ref(false)
const error = ref(null)

onMounted(() => {
  window.google.accounts.id.initialize({
    client_id: CLIENT_ID,
    callback: handleCredentialResponse,
  })
  renderGoogleBtn()
})

function renderGoogleBtn() {
  window.google.accounts.id.renderButton(
    document.getElementById('google-btn'),
    { theme: 'outline', size: 'large', text: 'signin_with' }
  )
}

function parseJwt(token) {
  const base64 = token.split('.')[1].replace(/-/g, '+').replace(/_/g, '/')
  return JSON.parse(atob(base64))
}

async function handleCredentialResponse(response) {
  idToken.value = response.credential
  user.value = parseJwt(response.credential)
  error.value = null
  apiResponse.value = null
  await postToApi(response.credential)
}

async function postToApi(token) {
  loading.value = true
  try {
    const res = await fetch('/api/v1/register/google', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ id_token: token }),
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
  window.google.accounts.id.disableAutoSelect()
  user.value = null
  idToken.value = null
  apiResponse.value = null
  error.value = null
}

function switchTab(next) {
  tab.value = next
  if (next === 'google' && !user.value) {
    // Re-render Google button after tab switch
    setTimeout(() => {
      const el = document.getElementById('google-btn')
      if (el && !el.firstChild) renderGoogleBtn()
    }, 0)
  }
}
</script>

<template>
  <div class="page">
    <div class="card">
      <h1>OAuth Login Sample</h1>

      <div class="tabs">
        <button :class="['tab', { active: tab === 'google' }]" @click="switchTab('google')">
          Google
        </button>
        <button :class="['tab', { active: tab === 'telegram' }]" @click="switchTab('telegram')">
          Telegram
        </button>
      </div>

      <!-- Google Tab -->
      <div v-show="tab === 'google'">
        <template v-if="!user">
          <p class="hint">Sign in with your Google account to test the OAuth flow.</p>
          <div id="google-btn"></div>
        </template>

        <template v-else>
          <div class="profile">
            <img :src="user.picture" :alt="user.name" class="avatar" />
            <div>
              <p class="name">{{ user.name }}</p>
              <p class="email">{{ user.email }}</p>
            </div>
          </div>

          <section class="section">
            <h2>ID Token (JWT)</h2>
            <textarea readonly :value="idToken" rows="4"></textarea>
          </section>

          <section class="section">
            <h2>Decoded Payload</h2>
            <pre>{{ JSON.stringify(user, null, 2) }}</pre>
          </section>

          <section class="section">
            <h2>API Response <code>POST /api/v1/register/google</code></h2>
            <div v-if="loading" class="loading">Sending to API…</div>
            <div v-else-if="error" class="error">{{ error }}</div>
            <pre v-else-if="apiResponse">Status: {{ apiResponse.status }}
{{ apiResponse.body }}</pre>
          </section>

          <button class="sign-out" @click="signOut">Sign out</button>
        </template>
      </div>

      <!-- Telegram Tab -->
      <div v-show="tab === 'telegram'">
        <TelegramLogin />
      </div>
    </div>
  </div>
</template>

<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: Inter, system-ui, sans-serif;
  background: #f0f2f5;
  min-height: 100vh;
}

.page {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  padding: 3rem 1rem;
  min-height: 100vh;
}

.card {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 4px 24px rgba(0,0,0,.08);
  padding: 2.5rem 2rem;
  width: 100%;
  max-width: 560px;
}

h1 {
  font-size: 1.5rem;
  font-weight: 700;
  margin-bottom: 1.5rem;
  color: #1a1a1a;
}

h2 {
  font-size: .85rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: .05em;
  color: #666;
  margin-bottom: .5rem;
}

.tabs {
  display: flex;
  gap: .5rem;
  margin-bottom: 1.75rem;
  border-bottom: 2px solid #e8e8e8;
  padding-bottom: 0;
}

.tab {
  padding: .5rem 1.25rem;
  background: none;
  border: none;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  font-size: .95rem;
  font-weight: 500;
  color: #888;
  cursor: pointer;
  transition: color .15s, border-color .15s;
}

.tab.active {
  color: #1a1a1a;
  border-bottom-color: #1a1a1a;
}

.tab:hover:not(.active) { color: #444; }

.hint {
  color: #555;
  margin-bottom: 1.5rem;
}

#google-btn {
  display: flex;
  justify-content: center;
}

.profile {
  display: flex;
  align-items: center;
  gap: 1rem;
  background: #f8f9fa;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1.5rem;
}

.avatar {
  width: 56px;
  height: 56px;
  border-radius: 50%;
  flex-shrink: 0;
}

.name { font-weight: 600; font-size: 1rem; }
.email { color: #666; font-size: .9rem; }

.section {
  margin-bottom: 1.5rem;
}

textarea, pre {
  width: 100%;
  background: #f4f4f4;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  padding: .75rem;
  font-size: .8rem;
  font-family: 'Fira Code', monospace;
  color: #333;
  overflow-x: auto;
  white-space: pre-wrap;
  word-break: break-all;
}

textarea { resize: vertical; }

code {
  font-family: 'Fira Code', monospace;
  font-size: .8rem;
  background: #eee;
  padding: 1px 5px;
  border-radius: 4px;
}

.loading { color: #888; font-style: italic; }
.error { color: #c0392b; background: #fdecea; border-radius: 6px; padding: .75rem; }

.sign-out {
  margin-top: .5rem;
  padding: .5rem 1.25rem;
  background: #fff;
  border: 1px solid #d0d0d0;
  border-radius: 6px;
  cursor: pointer;
  font-size: .9rem;
  color: #333;
  transition: background .15s;
}

.sign-out:hover { background: #f5f5f5; }
</style>
