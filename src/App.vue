<script setup>
import AudioMerger from './components/AudioMerger.vue'
</script>

<template>
  <div class="app">
    <header>
      <h1>Audio Merger</h1>
      <p class="subtitle">ffmpeg.wasm を使って2つの音声ファイルを結合します</p>
    </header>
    <main>
      <AudioMerger />
    </main>
  </div>
</template>

<style>
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  background-color: #f5f5f7;
  color: #1d1d1f;
  line-height: 1.6;
}

.app {
  max-width: 720px;
  margin: 0 auto;
  padding: 40px 20px;
}

header {
  text-align: center;
  margin-bottom: 40px;
}

header h1 {
  font-size: 2rem;
  font-weight: 700;
  color: #1d1d1f;
}

.subtitle {
  margin-top: 8px;
  color: #6e6e73;
  font-size: 0.95rem;
}
</style>
