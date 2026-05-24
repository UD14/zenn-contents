---
title: "今日のAI勉強会でやったこと：Antigravity CLI と Twitter API v2"
emoji: "⌨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AntigravityCLI", "TwitterAPI", "Python", "tweepy", "AI"]
published: true
---

今日、AI勉強会で2つのテーマに取り組みました。記録として残しておきます。

---

## セッション①：Antigravity CLI を導入する

### Antigravity CLI とは

ターミナルから `agy` コマンドで AI エージェントを直接操作できるツールです。
今使っているAntigravity（このチャット画面）と同じエンジンを、ターミナルだけで動かすことができます。

デスクトップ版と何が違うのでしょうか。一言で言えば、「AIがコマンドをそのまま実行できる」という点に尽きます。
調べて、コードを書いて、実行する。その一連の流れが、ターミナルの中で完結するのです。

### インストール

```bash
curl -fsSL https://antigravity.google/cli/install.sh | bash
source ~/.zshrc
agy --version
# → 1.0.2
```

インストール自体は1コマンドで終わります。PATHを通してバージョンが出ればセットアップ完了です。

初回起動時はGoogleアカウントでのログインを求められ、ブラウザが自動で開きます。

### 使い方

```bash
agy
```

起動後はターミナルに話しかけるだけです。

```
> このフォルダのファイルを一覧にして
> README.mdを要約して
```

今日、実際にCLIを使って作ってみたのがこちらの特設サイトです。
[AI勉強会 2026.05.24 - Cosmic & Cyber Vision](https://antigravity-kappa-lyart.vercel.app/AIbenkyokai20260524.html)

画面をクリックするとネオンパーティクルが弾け飛ぶという、インタラクティブな演出が実装されています。

ちなみに本日のプログラム（トーク内容）は以下の通りでした。
- **AIエージェントの現在地と未来予測**
- **Canvas物理演算とシェーダーの基礎**
- **対話型実装のライブデモンストレーション**

このように、AIに指示を出すだけで複雑な表現を含んだWebサイトがターミナル上で組み上がる。コードを直接実行できる環境にAIがいるというのは、デスクトップ版のチャットとはまた別の、圧倒的な手触りがありました。

---

## セッション②：Twitter API v2 で自動投稿スクリプトを作る

### 構成

Python と `tweepy` ライブラリを使います。認証情報はコードに直書きせず、`.env` ファイルで管理することにしました。

```bash
pip install tweepy python-dotenv
```

### スクリプト

```python
import os
import tweepy
from dotenv import load_dotenv

load_dotenv()

client = tweepy.Client(
    consumer_key=os.getenv("TWITTER_API_KEY"),
    consumer_secret=os.getenv("TWITTER_API_SECRET"),
    access_token=os.getenv("TWITTER_ACCESS_TOKEN"),
    access_token_secret=os.getenv("TWITTER_ACCESS_TOKEN_SECRET"),
)

response = client.create_tweet(text="Hello World from Twitter API v2!")
print(f"投稿成功: https://x.com/i/web/status/{response.data['id']}")
```

### `.env` の設定

```env
TWITTER_API_KEY=your_api_key_here
TWITTER_API_SECRET=your_api_secret_here
TWITTER_ACCESS_TOKEN=your_access_token_here
TWITTER_ACCESS_TOKEN_SECRET=your_access_token_secret_here
```

`.gitignore` に `.env` を含めることは必須です。認証情報をGitに乗せることだけは絶対に避けなければなりません。

### ハマりポイント

今日実際に詰まった箇所を、正直に書き残しておきます。

**403エラー（Forbidden）**
App permissionsが「Read only」のまま投稿しようとすると出るエラーです。
Developer Portal でアプリの権限を「Read and Write」に変更する必要があるのですが、変更後にAccess Tokenを再発行しないと反映されません。私はこれを見落としていて、時間を溶かしました。

**402エラー（Payment Required）**
APIキーを正しく設定し、権限も直したのにこのエラーが出ました。
原因は、TwitterのAPIプランが「Pay Per Use（残高ゼロ）」になっていたためでした。
$5分のクレジットを購入したところ、即座に解消されてテスト投稿が通ったのを確認しています。

現時点でFreeプラン（月1,500件まで無料）への切り替えは未確認ですが、ひとまずスクリプトが動く状態になったことは確認できました。

---

技術的にはどちらも難しいものではないはずです。ただ、セットアップの手順が細かく、どこかひとつ見落とすと詰まってしまいます。
権限の設定、トークンの再発行、プランの確認。この3つが、APIが「動かない」原因の大半を占めているのだと思います。

次は、このスクリプトをAntigravity CLIと組み合わせて、記事を書いたら自動でツイートまで流れる仕組みにしてみたいと考えています。
