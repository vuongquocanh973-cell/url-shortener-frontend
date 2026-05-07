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
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&display=swap');

* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: 'Inter', Arial, sans-serif;
  background: #060d1f;
  color: #e2eaf8;
  min-height: 100vh;
  padding: 40px 20px;
  display: flex;              /* add this */
  justify-content: center;   /* add this */
}

.container {
  max-width: 800px;
  width: 100%;               /* add this */
  margin: 0 auto;
}
/* Brand */
h1 {
  text-align: center;
  font-size: 2rem;
  font-weight: 600;
  color: #fff;
  margin-bottom: 6px;
  letter-spacing: -0.5px;
}

/* Auth tabs */
.auth-tabs {
  display: flex;
  gap: 0;
  background: #0b1122;
  border: 1px solid #1a2d58;
  border-radius: 12px;
  padding: 4px;
  max-width: 280px;
  margin: 0 auto 24px;
}
.auth-tabs button {
  flex: 1;
  padding: 9px 0;
  border: none;
  border-radius: 9px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  font-family: inherit;
  transition: all 0.2s;
  background: transparent;
  color: #64748b;
}
.auth-tabs button.active {
  background: #1d4ed8;
  color: #fff;
}

/* Auth form card */
.auth-form {
  background: #0b1530;
  border: 1px solid #1a2d58;
  border-radius: 16px;
  padding: 28px;
  margin-bottom: 20px;
  position: relative;
  overflow: hidden;
}
.auth-form::before {
  content: '';
  position: absolute;
  top: 0; left: 0; right: 0;
  height: 2px;
  background: linear-gradient(90deg, #1d4ed8, #60a5fa, #1d4ed8);
}
.auth-form h2 {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 2px;
  text-transform: uppercase;
  color: #3b82f6;
  margin-bottom: 20px;
}
.auth-form input {
  width: 100%;
  padding: 12px 16px;
  background: #07102a;
  border: 1px solid #1a2d58;
  border-radius: 10px;
  color: #e2eaf8;
  font-size: 14px;
  font-family: inherit;
  margin-bottom: 12px;
  transition: border-color 0.2s;
  outline: none;
}
.auth-form input:focus { border-color: #3b82f6; }
.auth-form input::placeholder { color: #334155; }
.auth-form button {
  width: 100%;
  padding: 13px;
  background: #1d4ed8;
  color: #fff;
  border: none;
  border-radius: 10px;
  font-weight: 600;
  font-size: 15px;
  cursor: pointer;
  font-family: inherit;
  transition: background 0.2s;
}
.auth-form button:hover { background: #2563eb; }

/* Divider */
.divider {
  text-align: center;
  color: #334155;
  font-size: 13px;
  margin: 20px 0;
  display: flex;
  align-items: center;
  gap: 12px;
}
.divider::before, .divider::after {
  content: '';
  flex: 1;
  height: 1px;
  background: #1a2d58;
}

/* Header bar (logged in) */
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 28px;
}
.user-info {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #0b1530;
  border: 1px solid #1a2d58;
  border-radius: 99px;
  padding: 7px 14px 7px 8px;
  font-size: 13px;
}
.user-info span::before {
  content: '';
  display: inline-block;
  width: 24px; height: 24px;
  background: #1d4ed8;
  border-radius: 50%;
  margin-right: 6px;
  vertical-align: middle;
}
.logout-btn {
  padding: 7px 14px;
  background: #1e0f0f;
  border: 1px solid #4a1515;
  border-radius: 8px;
  color: #f87171;
  font-size: 13px;
  cursor: pointer;
  font-family: inherit;
  margin-left: 8px;
}

/* Limit info */
.limit-info {
  font-size: 12px;
  color: #64748b;
  margin-bottom: 20px;
}

/* URL input row */
.input-section { display: flex; gap: 10px; margin-bottom: 16px; }
input {
  flex: 1;
  padding: 13px 16px;
  background: #07102a;
  border: 1px solid #1a2d58;
  border-radius: 10px;
  color: #e2eaf8;
  font-size: 14px;
  font-family: inherit;
  outline: none;
  transition: border-color 0.2s;
}
input:focus { border-color: #3b82f6; }
input::placeholder { color: #334155; }

button {
  padding: 13px 22px;
  background: #1d4ed8;
  color: #fff;
  border: none;
  border-radius: 10px;
  font-weight: 600;
  font-size: 15px;
  cursor: pointer;
  font-family: inherit;
  transition: background 0.2s;
  white-space: nowrap;
}
button:hover:not(:disabled) { background: #2563eb; }
button:disabled { opacity: 0.4; cursor: not-allowed; }

/* Result */
.result {
  background: #07102a;
  border: 1px solid #1e3a6e;
  border-radius: 12px;
  padding: 16px 20px;
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}
.result p { font-size: 12px; color: #3b82f6; font-weight: 600; }
.result a { color: #60a5fa; font-size: 14px; flex: 1; }
.result button {
  padding: 7px 14px;
  background: #0f2d6e;
  border: 1px solid #1e3a6e;
  border-radius: 8px;
  color: #60a5fa;
  font-size: 13px;
  font-weight: 500;
}
.result button:hover { background: #163a85; }

/* Messages */
.error { color: #f87171; font-size: 13px; margin-top: 6px; }
.success { color: #34d399; font-size: 13px; margin-top: 6px; }

/* Table */
.table-section {
  background: #0b1530;
  border: 1px solid #1a2d58;
  border-radius: 16px;
  overflow: hidden;
  margin-top: 28px;
}
h2 {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 2px;
  text-transform: uppercase;
  color: #3b82f6;
  padding: 16px 20px;
  border-bottom: 1px solid #1a2d58;
  margin: 0;
}
table { width: 100%; border-collapse: collapse; }
th {
  padding: 12px 20px;
  text-align: left;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 1px;
  text-transform: uppercase;
  color: #334155;
  background: #07102a;
}
td {
  padding: 13px 20px;
  font-size: 13px;
  border-top: 1px solid #101e3a;
  max-width: 180px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
td a { color: #60a5fa; text-decoration: none; }
td:nth-child(3) {
  font-weight: 500;
  color: #93c5fd;
}
</style>