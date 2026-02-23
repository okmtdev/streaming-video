<script setup>
import { ref, computed } from 'vue'
import { Mp3Encoder } from '@breezystack/lamejs'
import { Muxer, ArrayBufferTarget } from 'mp4-muxer'

const merging = ref(false)
const mergeTime = ref(null)
const errorMessage = ref('')

const file1 = ref(null)
const file2 = ref(null)
const file1Name = ref('')
const file2Name = ref('')

const volume1 = ref(100)   // %: 0〜200
const volume2 = ref(100)
const silenceSec = ref(0)  // 秒: 0〜10

const mergedOutputs = ref([])

const CODEC_CONFIG = {
  mp3: { mime: 'audio/mpeg', label: 'MP3' },
  wav: { mime: 'audio/wav', label: 'WAV' },
  m4a: { mime: 'audio/mp4', label: 'M4A' },
}

function getExtension(filename) {
  return filename.split('.').pop().toLowerCase()
}

function onFile1Change(event) {
  const f = event.target.files[0]
  if (f) { file1.value = f; file1Name.value = f.name }
}

function onFile2Change(event) {
  const f = event.target.files[0]
  if (f) { file2.value = f; file2Name.value = f.name }
}

const hasM4aInput = computed(() => {
  const ext1 = file1.value ? getExtension(file1.value.name) : ''
  const ext2 = file2.value ? getExtension(file2.value.name) : ''
  return ext1 === 'm4a' || ext2 === 'm4a'
})

// ── エンコーダー ──────────────────────────────────────────

function encodeWAV(audioBuffer) {
  const numChannels = audioBuffer.numberOfChannels
  const sampleRate = audioBuffer.sampleRate
  const numSamples = audioBuffer.length
  const bytesPerSample = 2
  const dataSize = numSamples * numChannels * bytesPerSample
  const buffer = new ArrayBuffer(44 + dataSize)
  const view = new DataView(buffer)

  const writeString = (offset, str) => {
    for (let i = 0; i < str.length; i++) view.setUint8(offset + i, str.charCodeAt(i))
  }

  writeString(0, 'RIFF')
  view.setUint32(4, 36 + dataSize, true)
  writeString(8, 'WAVE')
  writeString(12, 'fmt ')
  view.setUint32(16, 16, true)
  view.setUint16(20, 1, true)
  view.setUint16(22, numChannels, true)
  view.setUint32(24, sampleRate, true)
  view.setUint32(28, sampleRate * numChannels * bytesPerSample, true)
  view.setUint16(32, numChannels * bytesPerSample, true)
  view.setUint16(34, 16, true)
  writeString(36, 'data')
  view.setUint32(40, dataSize, true)

  let offset = 44
  for (let i = 0; i < numSamples; i++) {
    for (let ch = 0; ch < numChannels; ch++) {
      const s = Math.max(-1, Math.min(1, audioBuffer.getChannelData(ch)[i]))
      view.setInt16(offset, Math.round(s * (s < 0 ? 0x8000 : 0x7fff)), true)
      offset += 2
    }
  }

  return new Blob([buffer], { type: 'audio/wav' })
}

function encodeMp3(audioBuffer) {
  const numChannels = audioBuffer.numberOfChannels
  const sampleRate = audioBuffer.sampleRate
  const numSamples = audioBuffer.length

  const encoder = new Mp3Encoder(numChannels, sampleRate, 192)
  const mp3Chunks = []

  const left = new Int16Array(numSamples)
  const right = numChannels > 1 ? new Int16Array(numSamples) : null

  const leftData = audioBuffer.getChannelData(0)
  for (let i = 0; i < numSamples; i++) {
    left[i] = Math.round(Math.max(-1, Math.min(1, leftData[i])) * 0x7fff)
  }
  if (right) {
    const rightData = audioBuffer.getChannelData(1)
    for (let i = 0; i < numSamples; i++) {
      right[i] = Math.round(Math.max(-1, Math.min(1, rightData[i])) * 0x7fff)
    }
  }

  const BLOCK_SIZE = 1152
  for (let i = 0; i < numSamples; i += BLOCK_SIZE) {
    const leftBlock = left.subarray(i, i + BLOCK_SIZE)
    const rightBlock = right ? right.subarray(i, i + BLOCK_SIZE) : null
    const chunk = right
      ? encoder.encodeBuffer(leftBlock, rightBlock)
      : encoder.encodeBuffer(leftBlock)
    if (chunk.length > 0) mp3Chunks.push(chunk)
  }

  const finalChunk = encoder.flush()
  if (finalChunk.length > 0) mp3Chunks.push(finalChunk)

  return new Blob(mp3Chunks, { type: 'audio/mpeg' })
}

async function encodeM4a(audioBuffer) {
  if (typeof AudioEncoder === 'undefined') {
    throw new Error('このブラウザは WebCodecs API (AudioEncoder) に未対応です')
  }

  const { numberOfChannels, sampleRate } = audioBuffer

  const target = new ArrayBufferTarget()
  const muxer = new Muxer({
    target,
    audio: { codec: 'aac', numberOfChannels, sampleRate },
    fastStart: 'in-memory',
  })

  await new Promise((resolve, reject) => {
    let encoder
    try {
      encoder = new AudioEncoder({
        output: (chunk, metadata) => {
          try { muxer.addAudioChunk(chunk, metadata) } catch (e) { reject(e) }
        },
        error: reject,
      })

      encoder.configure({
        codec: 'mp4a.40.2',
        numberOfChannels,
        sampleRate,
        bitrate: 192_000,
      })

      const FRAME_SIZE = 1024
      for (let i = 0; i < audioBuffer.length; i += FRAME_SIZE) {
        const frameLength = Math.min(FRAME_SIZE, audioBuffer.length - i)
        const frameData = new Float32Array(frameLength * numberOfChannels)
        for (let ch = 0; ch < numberOfChannels; ch++) {
          const channelData = audioBuffer.getChannelData(ch)
          for (let j = 0; j < frameLength; j++) {
            frameData[j * numberOfChannels + ch] = channelData[i + j]
          }
        }
        const audioData = new AudioData({
          format: 'f32-interleaved',
          sampleRate,
          numberOfFrames: frameLength,
          numberOfChannels,
          timestamp: Math.round((i / sampleRate) * 1_000_000),
          data: frameData,
        })
        encoder.encode(audioData)
        audioData.close()
      }

      encoder.flush().then(() => { encoder.close(); resolve() }).catch(reject)
    } catch (e) {
      reject(e)
    }
  })

  muxer.finalize()
  return new Blob([target.buffer], { type: 'audio/mp4' })
}

async function encodeToFormat(fmt, audioBuffer) {
  switch (fmt) {
    case 'wav': return encodeWAV(audioBuffer)
    case 'mp3': return encodeMp3(audioBuffer)
    case 'm4a': return encodeM4a(audioBuffer)
  }
}

// ── メイン処理 ──────────────────────────────────────────

async function mergeAudio() {
  if (!file1.value || !file2.value) {
    errorMessage.value = '2つの音声ファイルを選択してください'
    return
  }

  merging.value = true
  mergeTime.value = null
  errorMessage.value = ''

  for (const o of mergedOutputs.value) URL.revokeObjectURL(o.url)
  mergedOutputs.value = []

  const startTime = performance.now()

  try {
    const ext1 = getExtension(file1.value.name)
    const ext2 = getExtension(file2.value.name)
    const fmt1 = CODEC_CONFIG[ext1] ? ext1 : 'wav'
    const fmt2 = CODEC_CONFIG[ext2] ? ext2 : 'wav'
    const outputFormats = fmt1 === fmt2 ? [fmt1] : [fmt1, fmt2]

    const vol1Factor = volume1.value / 100
    const vol2Factor = volume2.value / 100
    const silence = Math.max(0, Math.min(10, isNaN(silenceSec.value) ? 0 : silenceSec.value))

    const audioCtx = new AudioContext()

    const [ab1, ab2] = await Promise.all([file1.value.arrayBuffer(), file2.value.arrayBuffer()])
    const [buf1, buf2] = await Promise.all([
      audioCtx.decodeAudioData(ab1),
      audioCtx.decodeAudioData(ab2),
    ])

    const numChannels = Math.max(buf1.numberOfChannels, buf2.numberOfChannels)
    const sampleRate = audioCtx.sampleRate
    const silenceSamples = silence > 0 ? Math.round(sampleRate * silence) : 0
    const totalLength = buf1.length + silenceSamples + buf2.length

    const merged = audioCtx.createBuffer(numChannels, totalLength, sampleRate)

    for (let ch = 0; ch < numChannels; ch++) {
      const out = merged.getChannelData(ch)
      const src1 = buf1.getChannelData(Math.min(ch, buf1.numberOfChannels - 1))
      const src2 = buf2.getChannelData(Math.min(ch, buf2.numberOfChannels - 1))

      // buf1 に音量を適用してコピー
      for (let i = 0; i < buf1.length; i++) {
        out[i] = src1[i] * vol1Factor
      }
      // 無音区間: createBuffer で 0 初期化済みのためスキップ

      // buf2 に音量を適用してコピー
      const offset = buf1.length + silenceSamples
      for (let i = 0; i < buf2.length; i++) {
        out[offset + i] = src2[i] * vol2Factor
      }
    }

    audioCtx.close()

    const newOutputs = []
    for (const fmt of outputFormats) {
      const blob = await encodeToFormat(fmt, merged)
      newOutputs.push({
        url: URL.createObjectURL(blob),
        filename: `output.${fmt}`,
        label: CODEC_CONFIG[fmt].label,
      })
    }
    mergedOutputs.value = newOutputs
  } catch (e) {
    errorMessage.value = `合成に失敗しました: ${e?.message ?? e}`
  } finally {
    mergeTime.value = ((performance.now() - startTime) / 1000).toFixed(2)
    merging.value = false
  }
}

function downloadOutput(output) {
  const a = document.createElement('a')
  a.href = output.url
  a.download = output.filename
  a.click()
}
</script>

<template>
  <div class="merger">
    <div class="main-ui">
      <!-- ファイル入力 + 音量 + 無音 -->
      <div class="file-inputs">
        <!-- ファイル 1 -->
        <div class="file-input-group">
          <label class="file-label">音声ファイル 1</label>
          <div class="file-drop" @click="$refs.input1.click()">
            <input ref="input1" type="file" accept=".mp3,.wav,.m4a" class="hidden-input" @change="onFile1Change" />
            <div v-if="file1Name" class="file-name">{{ file1Name }}</div>
            <div v-else class="file-placeholder">
              <span class="upload-icon">&#8593;</span>
              <span>mp3 / wav / m4a を選択</span>
            </div>
          </div>
          <div class="volume-control">
            <span class="volume-label">音量</span>
            <input type="range" min="0" max="200" step="1" v-model.number="volume1" class="volume-slider" :disabled="!file1" />
            <span class="volume-value">{{ volume1 }}%</span>
          </div>
        </div>

        <!-- 中央: + と無音設定 -->
        <div class="concat-middle">
          <span class="concat-plus">+</span>
          <div class="silence-control">
            <input
              type="number" min="0" max="10" step="0.1"
              v-model.number="silenceSec"
              class="silence-input"
              placeholder="0"
            />
            <span class="silence-unit">秒の無音</span>
          </div>
        </div>

        <!-- ファイル 2 -->
        <div class="file-input-group">
          <label class="file-label">音声ファイル 2</label>
          <div class="file-drop" @click="$refs.input2.click()">
            <input ref="input2" type="file" accept=".mp3,.wav,.m4a" class="hidden-input" @change="onFile2Change" />
            <div v-if="file2Name" class="file-name">{{ file2Name }}</div>
            <div v-else class="file-placeholder">
              <span class="upload-icon">&#8593;</span>
              <span>mp3 / wav / m4a を選択</span>
            </div>
          </div>
          <div class="volume-control">
            <span class="volume-label">音量</span>
            <input type="range" min="0" max="200" step="1" v-model.number="volume2" class="volume-slider" :disabled="!file2" />
            <span class="volume-value">{{ volume2 }}%</span>
          </div>
        </div>
      </div>

      <!-- M4A 入力時の注意 -->
      <p v-if="hasM4aInput" class="m4a-note">
        M4A エンコードには WebCodecs API が必要です
        <span class="m4a-note-browsers">（Chrome 94+ / Firefox 130+ / Safari 16.4+）</span>
      </p>

      <!-- 合成ボタン -->
      <button class="btn btn-primary merge-btn" :disabled="merging || !file1 || !file2" @click="mergeAudio">
        <span v-if="merging" class="spinner"></span>
        {{ merging ? '合成中...' : '音声を合成する' }}
      </button>

      <!-- 結果 -->
      <div v-if="mergedOutputs.length > 0" class="result-section">
        <div class="result-header">
          <h3>合成結果</h3>
          <span v-if="mergeTime !== null" class="time-badge">処理時間: {{ mergeTime }} 秒</span>
        </div>
        <div
          v-for="output in mergedOutputs"
          :key="output.filename"
          class="output-item"
          :class="{ 'output-item--multi': mergedOutputs.length > 1 }"
        >
          <span class="format-label">{{ output.label }}</span>
          <audio controls :src="output.url" class="audio-player"></audio>
          <button class="btn btn-secondary" @click="downloadOutput(output)">
            ダウンロード ({{ output.filename }})
          </button>
        </div>
      </div>
    </div>

    <!-- エラー -->
    <div v-if="errorMessage" class="error">{{ errorMessage }}</div>
  </div>
</template>

<style scoped>
.merger {
  background: #fff;
  border-radius: 16px;
  padding: 32px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
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
.btn:disabled { opacity: 0.5; cursor: not-allowed; }
.btn-primary { background-color: #28a745; color: #fff; }
.btn-primary:hover:not(:disabled) { background-color: #2db84d; }
.btn-secondary { background-color: #e8e8ed; color: #1d1d1f; }
.btn-secondary:hover:not(:disabled) { background-color: #d2d2d7; }

.spinner {
  display: inline-block;
  width: 16px;
  height: 16px;
  border: 2px solid rgba(255,255,255,0.3);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* ── ファイル入力エリア ── */
.file-inputs {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  margin-bottom: 24px;
}

.file-input-group { flex: 1; }

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
.file-drop:hover { border-color: #28a745; background-color: #f0fff4; }

.hidden-input { display: none; }
.file-name { font-weight: 500; color: #1d1d1f; word-break: break-all; }
.file-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  color: #86868b;
  font-size: 0.9rem;
}
.upload-icon { font-size: 1.5rem; font-weight: bold; }

/* ── 音量スライダー ── */
.volume-control {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 10px;
  padding: 0 2px;
}
.volume-label { font-size: 0.78rem; color: #86868b; white-space: nowrap; }
.volume-slider { flex: 1; accent-color: #28a745; cursor: pointer; }
.volume-slider:disabled { opacity: 0.35; cursor: not-allowed; }
.volume-value {
  font-size: 0.8rem; font-weight: 600; color: #1d1d1f;
  min-width: 38px; text-align: right;
}

/* ── 中央: + と無音設定 ── */
.concat-middle {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
  padding-top: 30px;
  flex-shrink: 0;
}
.concat-plus { font-size: 2rem; font-weight: 700; color: #86868b; }
.silence-control { display: flex; flex-direction: column; align-items: center; gap: 3px; }
.silence-input {
  width: 60px;
  text-align: center;
  padding: 5px 6px;
  border: 1.5px solid #d2d2d7;
  border-radius: 8px;
  font-size: 0.9rem;
  color: #1d1d1f;
  outline: none;
  transition: border-color 0.2s;
}
.silence-input:focus { border-color: #28a745; }
.silence-unit { font-size: 0.74rem; color: #86868b; white-space: nowrap; }

/* ── M4A 注意 ── */
.m4a-note {
  margin-bottom: 16px;
  padding: 8px 12px;
  background: #fffbe6;
  border: 1px solid #ffe58f;
  border-radius: 8px;
  font-size: 0.82rem;
  color: #7a5c00;
}
.m4a-note-browsers { opacity: 0.75; }

/* ── 合成ボタン ── */
.merge-btn { display: block; width: 100%; justify-content: center; }

/* ── 結果 ── */
.result-section { margin-top: 24px; padding: 20px; background: #f5f5f7; border-radius: 12px; }
.result-header { display: flex; align-items: center; gap: 12px; margin-bottom: 16px; }
.result-header h3 { font-size: 1rem; margin: 0; }
.time-badge {
  font-size: 0.8rem; font-weight: 600; color: #fff;
  background: #28a745; padding: 3px 10px; border-radius: 20px;
}
.output-item { display: flex; flex-direction: column; gap: 10px; }
.output-item--multi { padding: 14px; background: #fff; border-radius: 10px; margin-bottom: 10px; }
.output-item--multi:last-child { margin-bottom: 0; }
.format-label {
  font-size: 0.78rem; font-weight: 700; color: #28a745;
  background: #e6f9ec; padding: 2px 10px; border-radius: 20px; align-self: flex-start;
}
.audio-player { width: 100%; }

/* ── エラー ── */
.error {
  margin-top: 16px; padding: 12px 16px; background: #fff2f2;
  color: #d70015; border-radius: 10px; font-size: 0.9rem; font-weight: 500;
}

@media (max-width: 600px) {
  .file-inputs { flex-direction: column; }
  .concat-middle { flex-direction: row; padding-top: 0; gap: 12px; }
}
</style>
