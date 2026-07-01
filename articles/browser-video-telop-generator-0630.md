---
title: "iPhoneで撮った動画に、ブラウザだけでテロップを焼く仕組みを作った話（＋Groq APIで爆速化）"
emoji: "🎬"
type: "tech"
topics: ["nextjs", "ffmpeg", "whisper", "wasm", "vercel", "groq"]
published: false
---

## きっかけ

業務用の短い動画を、もっと手軽に作れないか——という相談がきっかけでした。

「1〜2分の動画で、企業の人が撮ってアップするだけで、ある程度のクオリティになるもの。凝った編集は要らない。シンプルなテロップが入ればいい。」

要件は明確で、逆に言えばそれ以上のことは求められていません。であれば、可能な限りシンプルな仕組みで完結させたいと考えました。

ただ、あくまでプロトタイプ（モック）としての検証だったため、いくつかの前提がありました。
- サーバーサイドで重い動画エンコードを走らせる余裕はない（Vercelの無料枠を想定）
- コストをかけず、サクッと手元で動くものを試したい
- ただし、**文字起こしのスピードと精度は妥協したくない**

この前提を満たす方法を考えた結果、「動画処理はブラウザ内で完結させ、重い音声認識だけを超高速な外部APIに逃がす」というハイブリッドな結論に至りました。

## FFmpeg.wasm——動画編集はブラウザの中で

WebAssembly（WASM）という技術のおかげで、本来サーバーやローカル環境でしか動かせなかったFFmpegを、ブラウザ上で実行できるようになっています。`@ffmpeg/ffmpeg` というライブラリがそれにあたります。

実装自体は、Next.jsのクライアントコンポーネントからFFmpegインスタンスを初期化し、アップロードされた動画ファイルを仮想ファイルシステムに書き込んで加工する流れです。

動画はサーバーに一切送信されません。エンコード処理はすべてユーザーのブラウザ（端末のCPU）内で完結するため、Vercelの処理時間制限（Hobbyプランで10秒）にも引っかかりません。社内向けの動画など、外部に出したくない素材を扱う上でも、この構成は都合が良いと感じています。

## 文字起こしは「Groq API」で爆速処理

当初は文字起こしもブラウザ内（Transformers.js + Whisper-tiny）で完結させるアーキテクチャで組んでいました。しかし、ブラウザ内推論では「精度が低い」「初回ダウンロードが重い」「スマホだと処理落ちする」という課題に直面しました。

そこで白羽の矢が立ったのが、**Groq** という推論特化型プロセッサ（LPU）を提供するサービスです。
GroqのAPIを経由して `whisper-large-v3` モデルを叩くと、**数十秒の音声なら1〜2秒で、しかも人間並みの精度で**文字起こしが完了します。しかも無料枠（Free Tier）が非常に寛大です。

```typescript
// src/app/api/transcribe/route.ts
import { NextResponse } from 'next/server';

export async function POST(request: Request) {
  const formData = await request.formData();
  
  // GroqのWhisper API（完全互換）を叩く
  const response = await fetch('https://api.groq.com/openai/v1/audio/transcriptions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.GROQ_API_KEY}`,
    },
    body: formData,
  });

  const data = await response.json();
  return NextResponse.json({ text: data.text });
}
```
*※Next.jsのRoute Handlerをプロキシとして挟むことで、APIキーを隠蔽しています*

## カラオケ風テロップの動的生成（UXの改善）

Groqのおかげで完璧な長文テキストが即座に生成されるようになったのですが、次に「長い文章をそのまま動画に合成すると、画面が文字で埋め尽くされてしまう（または黒帯からはみ出る）」というUI上のバグが発生しました。

これを解決するため、FFmpegのコマンドを動的に生成し、**「話している長さに合わせて、1行ずつ次々と切り替わる」カラオケ風のテロップ**を実装しました。

```typescript
// 文字を12文字ずつのチャンクに分割
const maxLen = 12;
const lines = [];
for (let i = 0; i < telopText.length; i += maxLen) {
  lines.push(telopText.slice(i, i + maxLen));
}

// 動画の秒数を均等割りして、各チャンクの表示時間を計算
const timePerChunk = videoDuration / lines.length;

let drawtextFilters = '';
lines.forEach((line, index) => {
  const startTime = index * timePerChunk;
  const endTime = (index + 1) * timePerChunk;
  
  // Y座標は中央固定、enableオプションで指定時間のみ表示させる
  drawtextFilters += `,drawtext=fontfile=font.otf:text='${line}':fontcolor=white:fontsize=w/16:x=(w-text_w)/2:y=h-h/8-text_h/2:enable='between(t,${startTime},${endTime})'`;
});
```

この工夫により、縦型（iPhone等）の動画であっても、横幅基準でフォントサイズが自動計算され、絶対に文字が見切れない・重ならない、プロ顔負けのテロップシステムが完成しました。

## 技術構成のまとめ

最終的な構成は以下の通りです。

| レイヤー | 技術 | 備考 |
|---|---|---|
| フレームワーク | Next.js (App Router) | SSR/CSRのハイブリッド |
| 動画処理 | FFmpeg.wasm (`@ffmpeg/ffmpeg`) | クライアントサイド完結 |
| 音声認識 | **Groq API** (`whisper-large-v3`) | Route Handler経由で秘匿化 |
| 日本語フォント | Noto Sans CJK JP | `public/fonts/`に配置 |
| ホスティング | Vercel (Hobbyプラン) | 無料枠 |

## 振り返って

「動画編集をブラウザで行う（クライアントサイド）」＋「重いAI処理は最速のAPIに任せる（サーバーサイド）」という役割分担にしたことで、レスポンスの速さとサーバーコストゼロ（Vercel無料枠＋Groq無料枠）を両立することができました。

「動画を選んで、ボタンを押して、少し直して、ダウンロードする」
たったこれだけのステップで、テロップ付きの動画が仕上がる仕組みが動いています。まだまだ発展途上ですが、プロトタイプとしては十分すぎる成果が得られたと感じています。
