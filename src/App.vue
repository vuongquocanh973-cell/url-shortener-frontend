<template>
  <div class="container">
    <h1>🔗 URL Shortener</h1>

    <!-- Input Section -->
    <div class="input-section">
      <input
        v-model="originalUrl"
        type="text"
        placeholder="Paste your long URL here..."
        @keyup.enter="shortenUrl"
      />
      <button @click="shortenUrl" :disabled="loading">
        {{ loading ? 'Shortening...' : 'Shorten' }}
      </button>
    </div>

    <!-- Error Message -->
    <p v-if="error" class="error">{{ error }}</p>

    <!-- Result -->
    <div v-if="shortUrl" class="result">
      <p>✅ Your short URL:</p>
      <a :href="shortUrl" target="_blank">{{ shortUrl }}</a>
      <button @click="copyToClipboard">📋 Copy</button>
    </div>

    <!-- All URLs Table -->
    <div v-if="urls.length > 0" class="table-section">
      <h2>All Shortened URLs</h2>
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
     backendUrl: 'http://localhost:5000'
    }
  },
  mounted() {
    this.fetchAllUrls()
  },
  methods: {
    async shortenUrl() {
      if (!this.originalUrl) {
        this.error = 'Please enter a URL!'
        return
      }
      this.loading = true
      this.error = ''
      this.shortUrl = ''
      try {
        const response = await axios.post(
          `${this.backendUrl}/api/Url`,
          JSON.stringify(this.originalUrl),
          { headers: { 'Content-Type': 'application/json' } }
        )
        this.shortUrl = `${this.backendUrl}/${response.data.shortCode}`
        this.originalUrl = ''
        this.fetchAllUrls()
      } catch (err) {
        this.error = 'Something went wrong. Make sure your URL is valid!'
      }
      this.loading = false
    },
    async fetchAllUrls() {
      try {
        const response = await axios.get(`${this.backendUrl}/api/Url`)
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

.container {
  max-width: 800px;
  margin: 0 auto;
}

h1 {
  text-align: center;
  font-size: 2rem;
  margin-bottom: 30px;
  color: #38bdf8;
}

.input-section {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

input {
  flex: 1;
  padding: 12px 16px;
  border-radius: 8px;
  border: 1px solid #334155;
  background: #1e293b;
  color: #f1f5f9;
  font-size: 1rem;
}

button {
  padding: 12px 20px;
  background: #38bdf8;
  color: #0f172a;
  border: none;
  border-radius: 8px;
  font-weight: bold;
  cursor: pointer;
}

button:disabled { opacity: 0.5; cursor: not-allowed; }
button:hover:not(:disabled) { background: #7dd3fc; }

.error { color: #f87171; margin-bottom: 15px; }

.result {
  background: #1e293b;
  padding: 16px;
  border-radius: 8px;
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
}

.result a { color: #38bdf8; }

.table-section { margin-top: 30px; }
h2 { margin-bottom: 15px; color: #94a3b8; }

table { width: 100%; border-collapse: collapse; }
th, td { padding: 12px; text-align: left; border-bottom: 1px solid #334155; font-size: 0.9rem; }
th { background: #1e293b; color: #94a3b8; }
td a { color: #38bdf8; text-decoration: none; }
td { max-width: 200px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
</style>