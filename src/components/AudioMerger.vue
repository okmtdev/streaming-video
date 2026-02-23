<script setup>
import { ref, onMounted } from 'vue'
import { FFmpeg } from '@ffmpeg/ffmpeg'
import { fetchFile, toBlobURL } from '@ffmpeg/util'

const ffmpeg = new FFmpeg()

const loaded = ref(false)
const loading = ref(false)
const merging = ref(false)
const progress = ref(0)
const logMessages = ref([])
const errorMessage = ref('')

const file1 = ref(null)
const file2 = ref(null)
const file1Name = ref('')
const file2Name = ref('')

const mergedAudioUrl = ref('')
const mergedFileName = ref('')

const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.6/dist/umd'

async function loadFFmpeg() {
  loading.value = true
  errorMessage.value = ''
  try {
    ffmpeg.on('log', ({ message }) => {
      logMessages.value.push(message)
      if (logMessages.value.length > 50) {
        logMessages.value.shift()
      }
    })
    ffmpeg.on('progress', ({ progress: p }) => {
      progress.value = Math.round(p * 100)
    })
    await ffmpeg.load({
      coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
      wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm'),
    })
    loaded.value = true
  } catch (e) {
    errorMessage.value = `ffmpeg.wasm の読み込みに失敗しました: ${e.message}`
  } finally {
    loading.value = false
  }
}

function getExtension(filename) {
  return filename.split('.').pop().toLowerCase()
}

function onFile1Change(event) {
  const f = event.target.files[0]
  if (f) {
    file1.value = f
    file1Name.value = f.name
  }
}

function onFile2Change(event) {
  const f = event.target.files[0]
  if (f) {
    file2.value = f
    file2Name.value = f.name
  }
}

async function mergeAudio() {
  if (!file1.value || !file2.value) {
    errorMessage.value = '2つの音声ファイルを選択してください'
    return
  }

  merging.value = true
  progress.value = 0
  errorMessage.value = ''
  logMessages.value = []

  if (mergedAudioUrl.value) {
    URL.revokeObjectURL(mergedAudioUrl.value)
    mergedAudioUrl.value = ''
  }

  try {
    const ext1 = getExtension(file1.value.name)
    const ext2 = getExtension(file2.value.name)
    const inputFile1 = `input1.${ext1}`
    const inputFile2 = `input2.${ext2}`
    const outputFile = 'output.mp3'

    await ffmpeg.writeFile(inputFile1, await fetchFile(file1.value))
    await ffmpeg.writeFile(inputFile2, await fetchFile(file2.value))

    // Create a concat list file for ffmpeg
    // First convert both inputs to a common format (PCM WAV) then concatenate
    await ffmpeg.exec(['-i', inputFile1, '-acodec', 'pcm_s16le', '-ar', '44100', '-ac', '2', 'tmp1.wav'])
    await ffmpeg.exec(['-i', inputFile2, '-acodec', 'pcm_s16le', '-ar', '44100', '-ac', '2', 'tmp2.wav'])

    // Create concat file list
    const concatList = "file 'tmp1.wav'\nfile 'tmp2.wav'\n"
    await ffmpeg.writeFile('filelist.txt', new TextEncoder().encode(concatList))

    // Concatenate using concat demuxer and encode to mp3
    await ffmpeg.exec([
      '-f', 'concat',
      '-safe', '0',
      '-i', 'filelist.txt',
      '-acodec', 'libmp3lame',
      '-b:a', '192k',
      outputFile,
    ])

    const data = await ffmpeg.readFile(outputFile)
    const blob = new Blob([data.buffer], { type: 'audio/mpeg' })
    mergedAudioUrl.value = URL.createObjectURL(blob)
    mergedFileName.value = outputFile

    // Clean up temporary files in ffmpeg FS
    await ffmpeg.deleteFile(inputFile1)
    await ffmpeg.deleteFile(inputFile2)
    await ffmpeg.deleteFile('tmp1.wav')
    await ffmpeg.deleteFile('tmp2.wav')
    await ffmpeg.deleteFile('filelist.txt')
    await ffmpeg.deleteFile(outputFile)
  } catch (e) {
    errorMessage.value = `合成に失敗しました: ${e.message}`
  } finally {
    merging.value = false
  }
}

function downloadMerged() {
  if (!mergedAudioUrl.value) return
  const a = document.createElement('a')
  a.href = mergedAudioUrl.value
  a.download = mergedFileName.value
  a.click()
}
</script>

<template>
  <div class="merger">
    <!-- Load ffmpeg -->
    <div v-if="!loaded" class="load-section">
      <button class="btn btn-primary" :disabled="loading" @click="loadFFmpeg">
        <span v-if="loading" class="spinner"></span>
        {{ loading ? 'ffmpeg.wasm を読み込み中...' : 'ffmpeg.wasm を読み込む' }}
      </button>
      <p class="hint">初回は WASM ファイルのダウンロードに時間がかかります</p>
    </div>

    <!-- Main UI -->
    <div v-else class="main-ui">
      <!-- File inputs -->
      <div class="file-inputs">
        <div class="file-input-group">
          <label class="file-label">音声ファイル 1</label>
          <div class="file-drop" @click="$refs.input1.click()">
            <input
              ref="input1"
              type="file"
              accept=".mp3,.wav"
              class="hidden-input"
              @change="onFile1Change"
            />
            <div v-if="file1Name" class="file-name">{{ file1Name }}</div>
            <div v-else class="file-placeholder">
              <span class="upload-icon">&#8593;</span>
              <span>mp3 / wav ファイルを選択</span>
            </div>
          </div>
        </div>

        <div class="concat-icon">+</div>

        <div class="file-input-group">
          <label class="file-label">音声ファイル 2</label>
          <div class="file-drop" @click="$refs.input2.click()">
            <input
              ref="input2"
              type="file"
              accept=".mp3,.wav"
              class="hidden-input"
              @change="onFile2Change"
            />
            <div v-if="file2Name" class="file-name">{{ file2Name }}</div>
            <div v-else class="file-placeholder">
              <span class="upload-icon">&#8593;</span>
              <span>mp3 / wav ファイルを選択</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Merge button -->
      <button
        class="btn btn-primary merge-btn"
        :disabled="merging || !file1 || !file2"
        @click="mergeAudio"
      >
        <span v-if="merging" class="spinner"></span>
        {{ merging ? '合成中...' : '音声を合成する' }}
      </button>

      <!-- Progress -->
      <div v-if="merging" class="progress-section">
        <div class="progress-bar">
          <div class="progress-fill" :style="{ width: progress + '%' }"></div>
        </div>
        <span class="progress-text">{{ progress }}%</span>
      </div>

      <!-- Result -->
      <div v-if="mergedAudioUrl" class="result-section">
        <h3>合成結果</h3>
        <audio controls :src="mergedAudioUrl" class="audio-player"></audio>
        <button class="btn btn-secondary" @click="downloadMerged">
          ダウンロード ({{ mergedFileName }})
        </button>
      </div>

      <!-- Log -->
      <details v-if="logMessages.length > 0" class="log-section">
        <summary>ffmpeg ログ</summary>
        <pre class="log-content"><code>{{ logMessages.join('\n') }}</code></pre>
      </details>
    </div>

    <!-- Error -->
    <div v-if="errorMessage" class="error">
      {{ errorMessage }}
    </div>
  </div>
</template>

<style scoped>
.merger {
  background: #fff;
  border-radius: 16px;
  padding: 32px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
}

.load-section {
  text-align: center;
  padding: 40px 0;
}

.hint {
  margin-top: 12px;
  font-size: 0.85rem;
  color: #86868b;
}

.btn {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 12px 24px;
  border: none;
  border-radius: 10px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s, opacity 0.2s;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background-color: #0071e3;
  color: #fff;
}

.btn-primary:hover:not(:disabled) {
  background-color: #0077ed;
}

.btn-secondary {
  background-color: #e8e8ed;
  color: #1d1d1f;
}

.btn-secondary:hover:not(:disabled) {
  background-color: #d2d2d7;
}

.spinner {
  display: inline-block;
  width: 16px;
  height: 16px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.file-inputs {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 24px;
}

.file-input-group {
  flex: 1;
}

.file-label {
  display: block;
  font-weight: 600;
  font-size: 0.9rem;
  margin-bottom: 8px;
  color: #1d1d1f;
}

.file-drop {
  border: 2px dashed #d2d2d7;
  border-radius: 12px;
  padding: 24px 16px;
  text-align: center;
  cursor: pointer;
  transition: border-color 0.2s, background-color 0.2s;
}

.file-drop:hover {
  border-color: #0071e3;
  background-color: #f0f5ff;
}

.hidden-input {
  display: none;
}

.file-name {
  font-weight: 500;
  color: #1d1d1f;
  word-break: break-all;
}

.file-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  color: #86868b;
  font-size: 0.9rem;
}

.upload-icon {
  font-size: 1.5rem;
  font-weight: bold;
}

.concat-icon {
  font-size: 2rem;
  font-weight: 700;
  color: #86868b;
  padding-top: 24px;
}

.merge-btn {
  display: block;
  width: 100%;
  text-align: center;
  justify-content: center;
}

.progress-section {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 16px;
}

.progress-bar {
  flex: 1;
  height: 8px;
  background: #e8e8ed;
  border-radius: 4px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: #0071e3;
  border-radius: 4px;
  transition: width 0.3s ease;
}

.progress-text {
  font-size: 0.85rem;
  font-weight: 600;
  color: #6e6e73;
  min-width: 40px;
}

.result-section {
  margin-top: 24px;
  padding: 20px;
  background: #f5f5f7;
  border-radius: 12px;
}

.result-section h3 {
  font-size: 1rem;
  margin-bottom: 12px;
}

.audio-player {
  width: 100%;
  margin-bottom: 12px;
}

.log-section {
  margin-top: 20px;
}

.log-section summary {
  cursor: pointer;
  font-size: 0.85rem;
  color: #6e6e73;
  font-weight: 600;
}

.log-content {
  margin-top: 8px;
  padding: 12px;
  background: #1d1d1f;
  color: #a1ffa1;
  border-radius: 8px;
  font-size: 0.75rem;
  max-height: 200px;
  overflow-y: auto;
  white-space: pre-wrap;
  word-break: break-all;
}

.error {
  margin-top: 16px;
  padding: 12px 16px;
  background: #fff2f2;
  color: #d70015;
  border-radius: 10px;
  font-size: 0.9rem;
  font-weight: 500;
}

@media (max-width: 600px) {
  .file-inputs {
    flex-direction: column;
  }
  .concat-icon {
    padding-top: 0;
  }
}
</style>
