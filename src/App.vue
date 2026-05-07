<template>
  <div class="container">
    <!-- NOT LOGGED IN: Show Login/Register -->
    <div v-if="!isLoggedIn">
      <h1>🔗 URL Shortener</h1>

      <!-- Toggle between Login and Register -->
      <div class="auth-tabs">
        <button @click="authMode = 'login'" :class="{ active: authMode === 'login' }">Login</button>
        <button @click="authMode = 'register'" :class="{ active: authMode === 'register' }">Register</button>
      </div>

      <!-- Login Form -->
      <div v-if="authMode === 'login'" class="auth-form">
        <h2>Login</h2>
        <input v-model="authForm.username" type="text" placeholder="Username" />
        <input v-model="authForm.password" type="password" placeholder="Password" />
        <button @click="handleLogin">Login</button>
        <p v-if="authError" class="error">{{ authError }}</p>
      </div>

      <!-- Register Form -->
      <div v-if="authMode === 'register'" class="auth-form">
        <h2>Register</h2>
        <input v-model="authForm.username" type="text" placeholder="Username" />
        <input v-model="authForm.password" type="password" placeholder="Password" />
        <button @click="handleRegister">Register</button>
        <p v-if="authError" class="error">{{ authError }}</p>
        <p v-if="authSuccess" class="success">{{ authSuccess }}</p>
      </div>

      <!-- Guest URL Shortener -->
      <div class="divider">or continue as guest (max 2 URLs)</div>
      <div class="input-section">
        <input v-model="originalUrl" type="text" placeholder="Paste your long URL here..." @keyup.enter="shortenUrl" />
        <button @click="shortenUrl" :disabled="loading">{{ loading ? 'Shortening...' : 'Shorten' }}</button>
      </div>
      <p v-if="error" class="error">{{ error }}</p>
      <div v-if="shortUrl" class="result">
        <p>✅ Your short URL:</p>
        <a :href="shortUrl" target="_blank">{{ shortUrl }}</a>
        <button @click="copyToClipboard">📋 Copy</button>
      </div>
    </div>

    <!-- LOGGED IN: Show URL Shortener -->
    <div v-if="isLoggedIn">
      <div class="header">
        <h1>🔗 URL Shortener</h1>
        <div class="user-info">
          <span>👤 {{ username }}</span>
          <button @click="handleLogout" class="logout-btn">Logout</button>
        </div>
      </div>

      <p class="limit-info">You can shorten up to <strong>6 URLs</strong></p>

      <div class="input-section">
        <input v-model="originalUrl" type="text" placeholder="Paste your long URL here..." @keyup.enter="shortenUrl" />
        <button @click="shortenUrl" :disabled="loading">{{ loading ? 'Shortening...' : 'Shorten' }}</button>
      </div>

      <p v-if="error" class="error">{{ error }}</p>

      <div v-if="shortUrl" class="result">
        <p>✅ Your short URL:</p>
        <a :href="shortUrl" target="_blank">{{ shortUrl }}</a>
        <button @click="copyToClipboard">📋 Copy</button>
      </div>

      <div v-if="urls.length > 0" class="table-section">
        <h2>Your Shortened URLs</h2>
        <table>
          <thead>
            <tr>
              <th>Original URL</th>
              <th>Short Code</th>
              <th>Clicks</th>
              <th>Created At</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="url in urls" :key="url.id">
              <td><a :href="url.originalUrl" target="_blank">{{ url.originalUrl }}</a></td>
              <td><a :href="backendUrl + '/' + url.shortCode" target="_blank">{{ url.shortCode }}</a></td>
              <td>{{ url.clickCount }}</td>
              <td>{{ new Date(url.createdAt).toLocaleDateString() }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  data() {
    return {
      originalUrl: '',
      shortUrl: '',
      urls: [],
      loading: false,
      error: '',
      backendUrl: 'https://url-shortener-backend-07pt.onrender.com',
      isLoggedIn: false,
      username: '',
      token: '',
      authMode: 'login',
      authForm: { username: '', password: '' },
      authError: '',
      authSuccess: ''
    }
  },
  mounted() {
    // Check if already logged in
    const savedToken = localStorage.getItem('token')
    const savedUsername = localStorage.getItem('username')
    if (savedToken) {
      this.token = savedToken
      this.username = savedUsername
      this.isLoggedIn = true
      this.fetchAllUrls()
    }
  },
  methods: {
    async handleLogin() {
      this.authError = ''
      try {
        const response = await axios.post(`${this.backendUrl}/api/users/login`, {
          id: 0,
          username: this.authForm.username,
          passwordHash: this.authForm.password
        })
        this.token = response.data.token
        this.username = response.data.username
        localStorage.setItem('token', this.token)
        localStorage.setItem('username', this.username)
        this.isLoggedIn = true
        this.fetchAllUrls()
      } catch (err) {
        this.authError = err.response?.data?.message || 'Login failed!'
      }
    },

    async handleRegister() {
      this.authError = ''
      this.authSuccess = ''
      try {
        await axios.post(`${this.backendUrl}/api/users/register`, {
          id: 0,
          username: this.authForm.username,
          passwordHash: this.authForm.password
        })
        this.authSuccess = '✅ Registration successful! Please login.'
        this.authMode = 'login'
      } catch (err) {
        this.authError = err.response?.data?.message || 'Registration failed!'
      }
    },

    handleLogout() {
      this.isLoggedIn = false
      this.token = ''
      this.username = ''
      this.urls = []
      localStorage.removeItem('token')
      localStorage.removeItem('username')
    },

    async shortenUrl() {
      if (!this.originalUrl) {
        this.error = 'Please enter a URL!'
        return
      }
      this.loading = true
      this.error = ''
      this.shortUrl = ''
      try {
        const headers = this.token ? { Authorization: `Bearer ${this.token}` } : {}
        const response = await axios.post(
          `${this.backendUrl}/api/Url`,
          JSON.stringify(this.originalUrl),
          { headers: { 'Content-Type': 'application/json', ...headers } }
        )
        this.shortUrl = `${this.backendUrl}/${response.data.shortCode}`
        this.originalUrl = ''
        if (this.isLoggedIn) this.fetchAllUrls()
      } catch (err) {
        this.error = err.response?.data?.message || 'Something went wrong!'
      }
      this.loading = false
    },

    async fetchAllUrls() {
      try {
        const headers = this.token ? { Authorization: `Bearer ${this.token}` } : {}
        const response = await axios.get(`${this.backendUrl}/api/Url`, { headers })
        this.urls = response.data
      } catch (err) {
        console.error('Could not fetch URLs', err)
      }
    },

    copyToClipboard() {
      navigator.clipboard.writeText(this.shortUrl)
      alert('Copied to clipboard!')
    }
  }
}
</script>

<style>
* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: Arial, sans-serif;
  background: #0f172a;
  color: #f1f5f9;
  min-height: 100vh;
  padding: 40px 20px;
}

.container { max-width: 800px; margin: 0 auto; }

h1 { text-align: center; font-size: 2rem; margin-bottom: 30px; color: #38bdf8; }

.auth-tabs { display: flex; gap: 10px; margin-bottom: 20px; justify-content: center; }
.auth-tabs button { padding: 10px 30px; border: none; border-radius: 8px; cursor: pointer; background: #1e293b; color: #f1f5f9; font-size: 1rem; }
.auth-tabs button.active { background: #38bdf8; color: #0f172a; font-weight: bold; }

.auth-form { background: #1e293b; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
.auth-form h2 { margin-bottom: 15px; color: #38bdf8; }
.auth-form input { width: 100%; padding: 12px; margin-bottom: 10px; border-radius: 8px; border: 1px solid #334155; background: #0f172a; color: #f1f5f9; font-size: 1rem; }
.auth-form button { width: 100%; padding: 12px; background: #38bdf8; color: #0f172a; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; font-size: 1rem; }

.divider { text-align: center; margin: 20px 0; color: #94a3b8; font-size: 0.9rem; }

.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.user-info { display: flex; align-items: center; gap: 10px; }
.logout-btn { padding: 8px 16px; background: #ef4444; color: white; border: none; border-radius: 8px; cursor: pointer; }

.limit-info { text-align: center; color: #94a3b8; margin-bottom: 20px; }

.input-section { display: flex; gap: 10px; margin-bottom: 20px; }
input { flex: 1; padding: 12px 16px; border-radius: 8px; border: 1px solid #334155; background: #1e293b; color: #f1f5f9; font-size: 1rem; }
button { padding: 12px 20px; background: #38bdf8; color: #0f172a; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; }
button:disabled { opacity: 0.5; cursor: not-allowed; }
button:hover:not(:disabled) { background: #7dd3fc; }

.error { color: #f87171; margin-bottom: 15px; }
.success { color: #4ade80; margin-bottom: 15px; }

.result { background: #1e293b; padding: 16px; border-radius: 8px; margin-bottom: 20px; display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.result a { color: #38bdf8; }

.table-section { margin-top: 30px; }
h2 { margin-bottom: 15px; color: #94a3b8; }
table { width: 100%; border-collapse: collapse; }
th, td { padding: 12px; text-align: left; border-bottom: 1px solid #334155; font-size: 0.9rem; }
th { background: #1e293b; color: #94a3b8; }
td a { color: #38bdf8; text-decoration: none; }
td { max-width: 200px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
</style>