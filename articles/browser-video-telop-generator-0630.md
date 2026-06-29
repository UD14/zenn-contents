---
title: "iPhoneで撮った動画に、ブラウザだけでテロップを焼く仕組みを作った話"
emoji: "🎬"
type: "tech"
topics: ["nextjs", "ffmpeg", "whisper", "wasm", "vercel"]
published: false
---

## きっかけ

業務用の短い動画を、もっと手軽に作れないか——という相談がきっかけだった。

「1〜2分の動画で、型は5種類くらい。企業の人が撮ってアップするだけで、ある程度のクオリティになるもの。凝った編集は要らない。シンプルなテロップが入ればいい。」

要件は明確で、逆に言えばそれ以上のことは求められていない。であれば、可能な限りシンプルな仕組みで完結させたかった。

ただ、いくつかの制約があった。

- サーバーサイドで重い動画処理を走らせる余裕はない（Vercelの無料枠を想定していた）
- 担当者が触る以上、ローカルにFFmpegをインストールさせるわけにもいかない
- できれば、外部APIの従量課金も発生させたくない

この3つの制約を同時に満たす方法を考えた結果、「ブラウザの中ですべて完結させる」という結論に至った。

## FFmpeg.wasm——ブラウザの中で動画を編集する

WebAssembly（WASM）という技術のおかげで、本来サーバーやローカル環境でしか動かせなかったFFmpegを、ブラウザ上で実行できるようになっている。`@ffmpeg/ffmpeg` というライブラリがそれにあたる。

実装自体は、Next.jsのクライアントコンポーネントからFFmpegインスタンスを初期化し、アップロードされた動画ファイルを仮想ファイルシステムに書き込んで、`drawbox` と `drawtext` フィルタで加工する、という流れになる。

```typescript
// FFmpeg.wasmの初期化（クライアントサイドでのみ実行）
const ffmpeg = new FFmpeg();
const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.6/dist/umd';

await ffmpeg.load({
  coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript'),
  wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm'),
});
```

テロップの合成処理は、画面下部に半透明の黒帯を敷き、その中央にNoto Sans JPで白文字を描画するだけの、極めてシンプルなフィルタチェーンで構成している。

```typescript
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vf', `drawbox=y=ih-ih/5:color=black@0.6:width=iw:height=ih/5:t=fill,` +
         `drawtext=fontfile=font.otf:text='${text}':fontcolor=white:` +
         `fontsize=h/20:x=(w-text_w)/2:y=h-h/10-text_h/2`,
  '-c:v', 'libx264',
  '-c:a', 'copy',
  'output.mp4'
]);
```

動画はサーバーに一切送信されない。処理はすべてユーザーのブラウザ内で完結するため、Vercelの処理時間制限（Hobbyプランで10秒）にも引っかからない。社内向けの動画など、外部に出したくない素材を扱う上でも、この構成は都合が良かった。

## 文字起こしも、ブラウザの中で

テロップのテキストを毎回手入力するのは面倒だ。動画の中で話している内容を自動で文字に起こせないか。

ここでもサーバーを使わない方法を選んだ。`Transformers.js`（Hugging Faceが提供するブラウザ向けの推論ライブラリ）を使い、Whisperの軽量モデル（`Xenova/whisper-tiny`）をブラウザ上で直接動かしている。

```typescript
const { pipeline } = await import('@huggingface/transformers');
const transcriber = await pipeline(
  'automatic-speech-recognition',
  'Xenova/whisper-tiny'
);

const output = await transcriber(audioData, {
  language: 'japanese',
  task: 'transcribe',
});
```

精度は正直に言って、高くない。`whisper-tiny` はWhisperモデルの中で最も小さいものであり、日本語の認識精度は「だいたい合ってるけど、ところどころ怪しい」という水準になる。

ただ、今回の用途ではそれで十分だった。文字起こしの結果はいったんテキストエリアに表示され、担当者がその場で手直しできるようにしている。「完璧な自動化」ではなく「8割をAIが埋めて、残り2割を人間が直す」という半自動の設計にしたことで、実用上の問題はほとんどなくなった。

## iPhoneの動画で躓いた話

開発中、テスト用に生成したカラーバー動画では問題なく動いていたのだが、実際にiPhoneのインカメラで撮影した動画（`.mov`、HEVC/H.265コーデック）をアップロードした途端、文字起こしが失敗した。

原因は単純で、音声の抽出方法にあった。

当初は `Web Audio API` の `decodeAudioData()` に動画ファイルのArrayBufferを直接渡していたのだが、このAPIは多くのブラウザ環境で動画コンテナ形式のデコードに対応していない。音声ファイル（`.wav`や`.mp3`）であれば問題ないが、`.mov` や `.mp4` のような動画コンテナをそのまま渡すと静かに失敗する。

修正は、すでにロード済みの FFmpeg.wasm を活用する形にした。文字起こしの前段階として、まず動画から音声だけを WAV（16kHz、モノラル）に変換し、その WAV を `decodeAudioData()` に渡すようにしている。

```typescript
// FFmpegで動画から音声のみをWAVとして抽出
await ffmpeg.exec([
  '-i', inputFileName,
  '-vn',                // 映像を除外
  '-acodec', 'pcm_s16le', // PCM 16bit
  '-ar', '16000',       // 16kHz（Whisperの要求仕様）
  '-ac', '1',           // モノラル
  'extracted_audio.wav'
]);
```

この修正により、`.mov`、`.mp4`、`.webm` など、FFmpegが扱える動画形式であれば何でも対応できるようになった。テスト環境だけで検証を済ませてしまう危うさを、改めて実感した場面だった。

## 技術構成のまとめ

最終的な構成は以下の通り。

| レイヤー | 技術 |
|---|---|
| フレームワーク | Next.js (App Router) |
| 動画処理 | FFmpeg.wasm (`@ffmpeg/ffmpeg`) |
| 音声認識 | Transformers.js + Whisper-tiny (`Xenova/whisper-tiny`) |
| 日本語フォント | Noto Sans CJK JP（`public/fonts/`に配置） |
| ホスティング | Vercel (Hobbyプラン) |
| サーバーサイド処理 | なし（すべてクライアント完結） |

外部APIの呼び出しはゼロ。従量課金も発生しない。Vercelの無料枠で問題なく動作する。

## SSRとの戦い（補足）

Next.jsでこの構成を動かす際、少しだけ注意が必要な点があった。

FFmpeg.wasmは `new FFmpeg()` をモジュールスコープで呼び出すと、Next.jsのSSR（サーバーサイドレンダリング）時に `ffmpeg.wasm does not support nodejs` というエラーで落ちる。Transformers.jsも同様で、静的にimportするとビルド時に `onnxruntime-node` のネイティブバインディングを探しに行ってしまう。

対処としては、FFmpegのインスタンス化を `useEffect` 内に閉じ込め、Transformers.jsは `await import()` による動的インポートに変更した。ブラウザ専用のライブラリをNext.jsで扱う際の定番パターンではあるのだが、ビルドが通るまでに数回のトライアンドエラーを要した。

```typescript
// FFmpegのインスタンス化はuseEffect内で（SSR回避）
useEffect(() => {
  ffmpegRef.current = new FFmpeg();
  load();
}, []);

// Transformers.jsは動的インポートで（SSR回避）
const { pipeline, env } = await import('@huggingface/transformers');
```

## 振り返って

「ブラウザの中ですべて完結させる」という設計方針は、制約から生まれたものだった。サーバーにお金をかけられない、担当者に環境構築を求められない。その縛りの中で辿り着いた構成が、結果的にはセキュリティやプライバシーの面でも合理的なものになったのは、少し面白い。

一方で、Whisper-tinyの精度には限界がある。長い文章や専門用語が多い動画では、手直しの量が増えてしまう場面もあるだろう。モデルサイズを上げれば精度は改善するが、ブラウザでの初回ダウンロード時間とのトレードオフになる。

完璧な自動化からはほど遠い。でも、「動画を選んで、ボタンを押して、少し直して、ダウンロードする」——その4ステップでテロップ付きの動画が仕上がる仕組みが、ゼロ円で動いている。今のところは、それで十分だと思っている。
