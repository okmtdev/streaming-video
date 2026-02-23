# Audio Merger — ffmpeg.wasm vs Web Audio API 比較アプリ

## 目次

1. [リポジトリの概要](#1-リポジトリの概要)
2. [ffmpeg.wasm 版と Web Audio API 版の処理フローの違い](#2-ffmpegwasm-版と-web-audio-api-版の処理フローの違い)
3. [プロダクトに採用する場合の考慮点](#3-プロダクトに採用する場合の考慮点)
4. [ffmpeg.wasm の処理の詳細](#4-ffmpegwasm-の処理の詳細)
5. [Web Audio API の処理の詳細](#5-web-audio-api-の処理の詳細)
6. [コードの解説・読み方](#6-コードの解説読み方)
7. [Google Cloud Storage へのリリース手順](#7-google-cloud-storage-へのリリース手順)

---

## 1. リポジトリの概要

このリポジトリは、ブラウザ上で 2 つの音声ファイルを結合する Vue.js アプリです。
同じ機能を **2 種類のアプローチ** で実装し、それぞれの処理時間・品質・使い勝手を並べて比較できます。

### 実装されている 2 つの方式

| | ffmpeg.wasm 版 | Web Audio API 版 |
|---|---|---|
| **技術** | C 製 FFmpeg を WebAssembly で実行 | ブラウザ標準 API のみ |
| **初回ロード** | 必要（WASM ファイル約 30MB をダウンロード） | 不要・即時使用可 |
| **表示色** | 青 (WASM バッジ) | 緑 (Native バッジ) |

### 確認できること

- **入力形式**: MP3 / WAV / M4A の 3 形式に対応
- **出力形式の自動決定**: 入力が同じ形式 → 同形式で出力、異なる形式（例: MP3 + WAV）→ 両形式で出力
- **処理時間の計測**: ファイル書き込み〜エンコード完了までの全ステップを含む合計時間
- **音量調整**: ファイルごとに 0〜200% の範囲でスライダー調整
- **無音挿入**: 2 ファイルの間に 0〜10 秒の無音を挿入
- **ブラウザ完結**: サーバーへのアップロードなし、すべてクライアントサイドで処理

---

## 2. ffmpeg.wasm 版と Web Audio API 版の処理フローの違い

### ffmpeg.wasm 版の処理フロー

```
[ユーザーが2ファイルを選択]
        ↓
[ffmpeg.wasm を WASM Worker にロード]  ← 初回のみ、約30MB
        ↓
[入力ファイルを WASM 仮想FS に書き込み]
        ↓
[FFmpeg: 各ファイルを PCM WAV に変換]  ← 音量フィルターも同時適用
   -af volume=X -acodec pcm_s16le -ar 44100 -ac 2
        ↓
[FFmpeg: 無音WAVを生成]  ← silenceSec > 0 の場合のみ
   -f lavfi -i anullsrc=... -t N
        ↓
[concat demuxer 用リストファイルを生成]
   filelist.txt: file 'tmp1.wav' / file 'silence.wav' / file 'tmp2.wav'
        ↓
[FFmpeg: ファイルを結合して combined.wav を生成]
   -f concat -safe 0 -i filelist.txt
        ↓
[FFmpeg: 出力形式ごとにエンコード]
   mp3  → libmp3lame 192kbps
   wav  → pcm_s16le
   m4a  → aac 192kbps
        ↓
[WASM 仮想FS から読み出し → Blob URL 生成]
        ↓
[ダウンロード / 再生]
```

### Web Audio API 版の処理フロー

```
[ユーザーが2ファイルを選択]
        ↓
[File.arrayBuffer() で生バイト列を読み込み]  ← 並列実行
        ↓
[AudioContext.decodeAudioData()]  ← ブラウザ内蔵デコーダで並列デコード
   → Float32 PCM の AudioBuffer に展開
        ↓
[新しい AudioBuffer を作成]
   長さ = buf1.length + silenceSamples + buf2.length
        ↓
[Float32 配列をチャンネルごとにコピー]  ← 音量係数を掛けながら
   out[i]        = src1[i] * vol1Factor
   out[offset+i] = src2[i] * vol2Factor
   （無音区間は createBuffer の 0 初期化に任せる）
        ↓
[エンコード]
   wav → 手書き RIFF ヘッダ + Int16 PCM 変換
   mp3 → @breezystack/lamejs (MP3 エンコーダ)
   m4a → WebCodecs AudioEncoder + mp4-muxer
        ↓
[Blob URL 生成 → ダウンロード / 再生]
```

### フローの主な違い

| 観点 | ffmpeg.wasm | Web Audio API |
|---|---|---|
| デコード | FFmpeg が WASM 内で処理 | ブラウザの `decodeAudioData()` |
| 中間形式 | WASM 仮想ファイルシステム上の WAV | メモリ上の `AudioBuffer` (Float32) |
| 音量適用 | `-af volume=X` フィルター | Float32 に係数を掛け算 |
| 無音生成 | `anullsrc` (lavfi ソース) | `createBuffer()` のゼロ初期化を利用 |
| MP3 エンコード | libmp3lame (C 実装) | lamejs (JS 実装) |
| M4A エンコード | libaac (C 実装) | WebCodecs `AudioEncoder` |
| スレッド | Web Worker + SharedArrayBuffer | メインスレッド (エンコードのみ) |

---

## 3. プロダクトに採用する場合の考慮点

### ffmpeg.wasm を採用する場合

**できること**
- FFmpeg が対応するほぼすべてのコーデック・コンテナを扱える（動画も含む）
- フィルターグラフで複雑な音声処理（正規化、EQ、エフェクトなど）が可能
- 出力品質が C 実装 libmp3lame / libaac に依存するため高品質
- ブラウザ差異の影響を受けにくい（WASM は標準化済み）

**できないこと・制限**
- `SharedArrayBuffer` を使用するため COOP/COEP ヘッダが必要
  ```
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp
  ```
  → CDN からのスクリプトや iframe 埋め込みに制約が生じる
- 初回ロードに約 30MB の WASM ファイルダウンロードが必要
- メモリ消費が大きい（ファイルを WASM ヒープ + JS 側の両方に持つ）
- WASM Worker が複数タブで共有されないため、タブが増えるほどメモリが増加

**向いているユースケース**
- 動画も扱う多機能エディタ
- 複雑なフィルター処理が必要なツール
- デスクトップ PWA（ヘッダ設定が容易な環境）

### Web Audio API を採用する場合

**できること**
- 追加ロードなしで即時使用可能
- 軽量（lamejs と mp4-muxer を合わせても数百 KB 未満）
- COOP/COEP ヘッダ不要
- `AudioContext` で正確なサンプルレートに自動正規化

**できないこと・制限**
- M4A エンコードに WebCodecs API (`AudioEncoder`) が必要
  → Safari 16.4+ / Chrome 94+ / Firefox 130+（古いブラウザは非対応）
- MP3 エンコードは JS 実装のため ffmpeg.wasm より遅い傾向
- 扱える形式がブラウザのデコーダ実装に依存する
- 長時間・大容量ファイルではメモリに全サンプルを展開するため OOM の可能性

**向いているユースケース**
- シンプルな音声結合・トリミング機能
- ヘッダ制御が難しい共有ホスティング環境
- モバイルファースト（ロード時間を最小化したい場合）

---

## 4. ffmpeg.wasm の処理の詳細

### WASM ロードの仕組み

```js
// vite.config.js: COOP/COEP ヘッダの設定
server: {
  headers: {
    'Cross-Origin-Opener-Policy': 'same-origin',
    'Cross-Origin-Embedder-Policy': 'require-corp',
  },
}
```

ffmpeg.wasm は内部で `SharedArrayBuffer` を使用します。`SharedArrayBuffer` はセキュリティ上の理由からクロスオリジン分離環境でのみ有効になるため、上記のヘッダが必須です。

```js
// AudioMerger.vue: WASM ファイルを Blob URL でラップしてロード
const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.6/dist/esm'

await ffmpeg.load({
  coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
  wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm'),
})
```

`toBlobURL` は外部 URL からファイルを fetch し、同一オリジンの Blob URL に変換します。これにより COEP 制約を満たしながら、`type: "module"` な Web Worker 内で ESM として動的 `import()` できます。

> **ESM 版を使う理由**: ffmpeg.wasm の Web Worker は `type: "module"` で動作するため `importScripts()` が使えません。`/umd/` ビルドは `this.createFFmpegCore` をグローバルに設定しますが、ESM スコープでは `this` が `undefined` になり失敗します。`/esm/` ビルドを使うことでこの問題を回避しています。

### mergeAudio() の全ステップ

**Step 1: 入力ファイルを WASM 仮想FS に書き込み**

```js
await ffmpeg.writeFile(`input1.${ext1}`, await fetchFile(file1.value))
await ffmpeg.writeFile(`input2.${ext2}`, await fetchFile(file2.value))
```

`fetchFile` は `File` オブジェクトを `Uint8Array` に変換します。WASM 内部はサンドボックス化された仮想ファイルシステム（Emscripten FS）を持っており、ここにファイルを「置く」ことで FFmpeg コマンドから参照できます。

**Step 2: 音量フィルター付きで PCM WAV に変換**

```js
await ffmpeg.exec([
  '-i', `input1.${ext1}`,
  '-af', `volume=${vol1}`,       // 例: volume=1.500
  '-acodec', 'pcm_s16le',        // 符号付き16bit リトルエンディアン PCM
  '-ar', '44100',                // サンプルレートを 44100Hz に統一
  '-ac', '2',                    // ステレオに統一
  'tmp1.wav'
])
```

中間形式を非圧縮 WAV (PCM) にすることで、連結時の再エンコードによる品質劣化を防いでいます。

**Step 3: 無音ファイルの生成**

```js
await ffmpeg.exec([
  '-f', 'lavfi',
  '-i', 'anullsrc=channel_layout=stereo:sample_rate=44100',
  '-t', String(silence),
  '-acodec', 'pcm_s16le',
  'silence.wav',
])
```

`lavfi` (libavfilter 仮想デバイス) の `anullsrc` ソースを使い、完全な無音 PCM を生成します。

**Step 4: concat demuxer で結合**

```
filelist.txt の内容:
file 'tmp1.wav'
file 'silence.wav'   ← silenceSec > 0 の場合のみ
file 'tmp2.wav'
```

```js
await ffmpeg.exec([
  '-f', 'concat', '-safe', '0',
  '-i', 'filelist.txt',
  '-acodec', 'pcm_s16le',
  'combined.wav'
])
```

`-safe 0` はリスト内のパスに相対パスを許可するフラグです。

**Step 5: 出力形式ごとにエンコード**

```js
const CODEC_CONFIG = {
  mp3: { args: ['-acodec', 'libmp3lame', '-b:a', '192k'], mime: 'audio/mpeg' },
  wav: { args: ['-acodec', 'pcm_s16le'], mime: 'audio/wav' },
  m4a: { args: ['-acodec', 'aac', '-b:a', '192k'], mime: 'audio/mp4' },
}
```

`combined.wav` を入力として各コーデックでエンコードし、`readFile()` で Uint8Array として取り出した後、`Blob` → `URL.createObjectURL()` でブラウザで再生・ダウンロード可能な URL を生成します。

### 処理時間の計測範囲

```js
const startTime = performance.now()   // ← 計測開始
try {
  // writeFile, exec × N, readFile, deleteFile ...
} finally {
  mergeTime.value = ((performance.now() - startTime) / 1000).toFixed(2)
}
```

`performance.now()` で全ステップ（ファイル書き込み・変換・連結・エンコード・読み出し）の合計時間を計測します。ffmpeg.wasm の `progress` イベントは FFmpeg 内部の進捗を示しますが、時間計測には `performance.now()` を使用しています。

---

## 5. Web Audio API の処理の詳細

### デコード

```js
const audioCtx = new AudioContext()

const [ab1, ab2] = await Promise.all([
  file1.value.arrayBuffer(),
  file2.value.arrayBuffer(),
])
const [buf1, buf2] = await Promise.all([
  audioCtx.decodeAudioData(ab1),
  audioCtx.decodeAudioData(ab2),
])
```

2 ファイルの読み込みとデコードを並列実行します。`decodeAudioData()` はブラウザのネイティブデコーダを使い、MP3・WAV・M4A すべてを **Float32 PCM の `AudioBuffer`** に展開します。サンプルレートは `AudioContext` のレートに自動正規化されます（通常 44100Hz または 48000Hz）。

### バッファ結合

```js
const silenceSamples = Math.round(sampleRate * silence)
const totalLength = buf1.length + silenceSamples + buf2.length
const merged = audioCtx.createBuffer(numChannels, totalLength, sampleRate)

for (let ch = 0; ch < numChannels; ch++) {
  const out = merged.getChannelData(ch)  // Float32Array への参照

  // ファイル1（音量適用）
  for (let i = 0; i < buf1.length; i++) {
    out[i] = src1[i] * vol1Factor
  }
  // 無音区間: createBuffer() はゼロ初期化されているためコピー不要

  // ファイル2（音量適用、オフセット付き）
  const offset = buf1.length + silenceSamples
  for (let i = 0; i < buf2.length; i++) {
    out[offset + i] = src2[i] * vol2Factor
  }
}
```

`createBuffer()` の戻り値は全サンプルが 0.0 で初期化されているため、無音区間のコピーは不要です。チャンネル数が異なる場合（例: モノラル + ステレオ）は最後のチャンネルを繰り返します。

### WAV エンコード (`encodeWAV`)

```
RIFF ヘッダ (44 バイト):
  "RIFF" + ファイルサイズ
  "WAVE"
  "fmt " チャンク: PCM, チャンネル数, サンプルレート, ビットレート, ビット深度(16)
  "data" チャンク: サンプルデータ

サンプルデータ (インターリーブ形式):
  Float32 → Int16 変換: Math.round(s * 0x7fff)
  正: 最大 +32767 (0x7fff)
  負: 最大 -32768 (0x8000)
```

標準ライブラリを使わず `DataView` で RIFF バイナリを直接構築しています。

### MP3 エンコード (`encodeMp3`)

```js
const encoder = new Mp3Encoder(numChannels, sampleRate, 192)  // 192kbps

const BLOCK_SIZE = 1152  // MPEG フレームサイズ (= 1152 サンプル)
for (let i = 0; i < numSamples; i += BLOCK_SIZE) {
  const chunk = encoder.encodeBuffer(leftBlock, rightBlock)
  if (chunk.length > 0) mp3Chunks.push(chunk)
}
const finalChunk = encoder.flush()  // 残余バッファのフラッシュ
```

MP3 は 1 フレームあたり 1152 サンプルを処理するため、BLOCK_SIZE を 1152 に設定しています。`@breezystack/lamejs` は lamejs の ESM 書き直し版で、Vite の ESM 環境でも `MPEGMode is not defined` エラーが発生しません（元の lamejs はグローバル変数参照を持つ CJS 実装のため ESM 環境では動作しない）。

### M4A エンコード (`encodeM4a`)

```js
// mp4-muxer でコンテナを作成
const target = new ArrayBufferTarget()
const muxer = new Muxer({
  target,
  audio: { codec: 'aac', numberOfChannels, sampleRate },
  fastStart: 'in-memory',  // moov atom を先頭に配置
})

// WebCodecs AudioEncoder でエンコード
const encoder = new AudioEncoder({
  output: (chunk, metadata) => muxer.addAudioChunk(chunk, metadata),
  error: reject,
})
encoder.configure({
  codec: 'mp4a.40.2',  // AAC-LC
  numberOfChannels,
  sampleRate,
  bitrate: 192_000,
})

// 1024 サンプルごとに AudioData を送信
const FRAME_SIZE = 1024  // AAC フレームサイズ
for (let i = 0; i < audioBuffer.length; i += FRAME_SIZE) {
  const audioData = new AudioData({
    format: 'f32-interleaved',
    timestamp: Math.round((i / sampleRate) * 1_000_000),  // マイクロ秒
    data: frameData,
  })
  encoder.encode(audioData)
  audioData.close()  // メモリ解放
}
await encoder.flush()
muxer.finalize()  // MP4 コンテナを完成
```

`mp4a.40.2` は AAC-LC (Low Complexity) の codec 文字列です。WebCodecs は非同期パイプラインのため `encoder.flush()` で全フレームの完了を待ちます。`mp4-muxer` の `fastStart: 'in-memory'` はメモリ上で moov atom を先頭配置し、シークしやすい MP4 を生成します。

---

## 6. コードの解説・読み方

### ファイル構成

```
streaming-video/
├── src/
│   ├── App.vue                    # レイアウト・セクション定義
│   ├── components/
│   │   ├── AudioMerger.vue        # ffmpeg.wasm 版（青）
│   │   └── WebAudioMerger.vue     # Web Audio API 版（緑）
│   └── main.js                    # Vue アプリのエントリポイント
├── vite.config.js                 # COEP/COOP ヘッダ設定、最適化除外
├── package.json
└── index.html
```

### App.vue

2 つのコンポーネントを並べて表示するだけのレイアウトファイルです。`<section>` 要素ごとにバッジ（WASM / Native）と説明文を付与しています。CSS は `<style>` にグローバルスコープで記述されています。

### AudioMerger.vue（ffmpeg.wasm 版）の読み方

| 箇所 | 内容 |
|---|---|
| `baseURL` | ESM ビルドの CDN URL（UMD では Worker で動作しない） |
| `CODEC_CONFIG` | 拡張子 → FFmpeg 引数・MIME タイプのマッピング |
| `loadFFmpeg()` | WASM ロード。`toBlobURL` で COEP を回避 |
| `ffmpeg.on('log', ...)` | FFmpeg の標準出力をキャプチャし UI に表示 |
| `ffmpeg.on('progress', ...)` | エンコード進捗（0〜1）をプログレスバーに反映 |
| `mergeAudio()` | メイン処理。Step 1〜6 のコメントで各 FFmpeg コマンドの意図を示している |
| `outputFormats` | `fmt1 === fmt2 ? [fmt1] : [fmt1, fmt2]` — 形式一致で 1 出力、不一致で 2 出力 |

### WebAudioMerger.vue（Web Audio API 版）の読み方

| 箇所 | 内容 |
|---|---|
| `encodeWAV()` | 同期関数。DataView で RIFF バイナリを直接書く |
| `encodeMp3()` | 同期関数。lamejs は同期エンコーダのため await 不要 |
| `encodeM4a()` | 非同期関数。WebCodecs は非同期パイプラインのため `Promise` でラップ |
| `encodeToFormat()` | `switch` で 3 つのエンコーダへのディスパッチ |
| `hasM4aInput` | `computed`。M4A ファイル選択時に WebCodecs の注意書きを表示 |
| `mergeAudio()` | `Promise.all` で 2 ファイルを並列デコード後、バッファを結合してエンコード |

### vite.config.js

```js
optimizeDeps: {
  exclude: ['@ffmpeg/ffmpeg', '@ffmpeg/util'],
}
```

ffmpeg.wasm は Vite の事前バンドル（esbuild によるモジュール変換）と競合するため、除外設定が必要です。バンドルされると Worker の動的 import が壊れます。
