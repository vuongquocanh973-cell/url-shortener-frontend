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
  background: #172448;
  color: #e8f0ff;
  min-height: 100vh;
  padding: 40px 20px;
  display: flex;
  justify-content: center;
}
 
.container {
  max-width: 800px;
  width: 100%;
  margin: 0 auto;
}
 
/* Brand */
h1 {
  text-align: center;
  font-size: 2rem;
  font-weight: 600;
  color: #fff;
  margin-bottom: 30px;
  letter-spacing: -0.5px;
}
 
/* Auth tabs */
.auth-tabs {
  display: flex;
  gap: 0;
  background: #1e3460;
  border: 1px solid #3a5a90;
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
  color: #93b4d8;
}
.auth-tabs button.active {
  background: #2563eb;
  color: #fff;
}
 
/* Auth form card */
.auth-form {
  background: #1e3868;
  border: 1px solid #3a5a90;
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
  background: linear-gradient(90deg, #2563eb, #93c5fd, #2563eb);
}
.auth-form h2 {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 2px;
  text-transform: uppercase;
  color: #93c5fd;
  margin-bottom: 20px;
}
.auth-form input {
  width: 100%;
  padding: 12px 16px;
  background: #162d58;
  border: 1px solid #3a5a90;
  border-radius: 10px;
  color: #e8f0ff;
  font-size: 14px;
  font-family: inherit;
  margin-bottom: 12px;
  transition: border-color 0.2s;
  outline: none;
}
.auth-form input:focus { border-color: #60a5fa; }
.auth-form input::placeholder { color: #6080a8; }
.auth-form button {
  width: 100%;
  padding: 13px;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 10px;
  font-weight: 600;
  font-size: 15px;
  cursor: pointer;
  font-family: inherit;
  transition: background 0.2s;
}
.auth-form button:hover { background: #3b82f6; }
 
/* Divider */
.divider {
  text-align: center;
  color: #6080a8;
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
  background: #3a5a90;
}
 
/* Header bar (logged in) */
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}
.header h1 {
  margin-bottom: 0;
  text-align: left;
}
.user-info {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #1e3868;
  border: 1px solid #3a5a90;
  border-radius: 99px;
  padding: 7px 14px;
  font-size: 13px;
  color: #93b4d8;
}
.logout-btn {
  padding: 7px 14px;
  background: #3a1a1a;
  border: 1px solid #6a2a2a;
  border-radius: 8px;
  color: #fca5a5;
  font-size: 13px;
  cursor: pointer;
  font-family: inherit;
  font-weight: 500;
  transition: background 0.2s;
}
.logout-btn:hover { background: #4a2020; }
 
/* Limit info */
.limit-info {
  font-size: 13px;
  color: #93b4d8;
  margin-bottom: 20px;
  text-align: center;
}
.limit-info strong { color: #93c5fd; }
 
/* URL input row */
.input-section { display: flex; gap: 10px; margin-bottom: 16px; }
 
input {
  flex: 1;
  padding: 13px 16px;
  background: #162d58;
  border: 1px solid #3a5a90;
  border-radius: 10px;
  color: #e8f0ff;
  font-size: 14px;
  font-family: inherit;
  outline: none;
  transition: border-color 0.2s;
}
input:focus { border-color: #60a5fa; }
input::placeholder { color: #6080a8; }
 
button {
  padding: 13px 22px;
  background: #2563eb;
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
button:hover:not(:disabled) { background: #3b82f6; }
button:disabled { opacity: 0.4; cursor: not-allowed; }
 
/* Result */
.result {
  background: #162d58;
  border: 1px solid #3a5a90;
  border-radius: 12px;
  padding: 16px 20px;
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}
.result p { font-size: 12px; color: #93c5fd; font-weight: 600; }
.result a { color: #93c5fd; font-size: 14px; flex: 1; }
.result button {
  padding: 7px 14px;
  background: #2a4a8a;
  border: 1px solid #3a5a90;
  border-radius: 8px;
  color: #93c5fd;
  font-size: 13px;
  font-weight: 500;
  width: auto;
}
.result button:hover { background: #3a5a9a; }
 
/* Messages */
.error { color: #fca5a5; font-size: 13px; margin-top: 8px; }
.success { color: #6ee7b7; font-size: 13px; margin-top: 8px; }
 
/* Table section */
.table-section {
  background: #1e3868;
  border: 1px solid #3a5a90;
  border-radius: 16px;
  overflow: hidden;
  margin-top: 28px;
  position: relative;
}
.table-section::before {
  content: '';
  position: absolute;
  top: 0; left: 0; right: 0;
  height: 2px;
  background: linear-gradient(90deg, #2563eb, #93c5fd, #2563eb);
}
 
h2 {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 2px;
  text-transform: uppercase;
  color: #93c5fd;
  padding: 16px 20px;
  border-bottom: 1px solid #3a5a90;
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
  color: #6080a8;
  background: #162d58;
}
 
td {
  padding: 13px 20px;
  font-size: 13px;
  border-top: 1px solid #2a4878;
  max-width: 200px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: #93b4d8;
}
 
td a { color: #93c5fd; text-decoration: none; }
td a:hover { color: #bfdbfe; }
</style>