<script setup>
import AudioMerger from './components/AudioMerger.vue'
import WebAudioMerger from './components/WebAudioMerger.vue'
</script>

<template>
  <div class="app">
    <header>
      <h1>Audio Merger</h1>
    </header>

    <main>
      <section class="section">
        <div class="section-header">
          <h2 class="section-title">ffmpeg.wasm 版</h2>
          <span class="badge badge-wasm">WASM</span>
        </div>
        <p class="section-desc">
          対応入力: mp3 / wav / m4a &nbsp;|&nbsp;
          出力: 入力と同形式（mp3+wav など異形式の場合は両方出力） &nbsp;|&nbsp;
          初回のみ WASM ファイルの読み込みが必要
        </p>
        <AudioMerger />
      </section>

      <div class="divider"></div>

      <section class="section">
        <div class="section-header">
          <h2 class="section-title">Web Audio API 版</h2>
          <span class="badge badge-web">Native</span>
        </div>
        <p class="section-desc">
          対応入力: mp3 / wav / m4a &nbsp;|&nbsp;
          出力: 入力と同形式（mp3+wav など異形式の場合は両方出力） &nbsp;|&nbsp;
          読み込み不要・即時使用可能
        </p>
        <WebAudioMerger />
      </section>
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
  padding: 40px 20px 80px;
}

header {
  text-align: center;
  margin-bottom: 48px;
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

.section {
  margin-bottom: 8px;
}

.section-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 8px;
}

.section-title {
  font-size: 1.15rem;
  font-weight: 700;
  color: #1d1d1f;
}

.badge {
  font-size: 0.72rem;
  font-weight: 700;
  padding: 3px 9px;
  border-radius: 20px;
  letter-spacing: 0.04em;
}

.badge-wasm {
  background: #e8f0fe;
  color: #0071e3;
}

.badge-web {
  background: #e6f9ec;
  color: #28a745;
}

.section-desc {
  font-size: 0.82rem;
  color: #6e6e73;
  margin-bottom: 16px;
}

.divider {
  display: flex;
  align-items: center;
  gap: 16px;
  margin: 40px 0;
}

.divider::before,
.divider::after {
  content: '';
  flex: 1;
  height: 1px;
  background: #d2d2d7;
}

.divider-label {
  font-size: 0.85rem;
  font-weight: 700;
  color: #86868b;
  padding: 4px 14px;
  border: 1px solid #d2d2d7;
  border-radius: 20px;
  background: #f5f5f7;
}
</style>
