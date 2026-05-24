---
title: "今日のAI勉強会でやったこと：Antigravity CLI と Twitter API v2"
emoji: "⌨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AntigravityCLI", "TwitterAPI", "Python", "tweepy", "AI"]
published: true
---

今日、AI勉強会で2つのことをやった。記録として残しておく。

---

## セッション①：Antigravity CLI を導入する

### Antigravity CLI とは

ターミナルから `agy` コマンドで AI エージェントを直接操作できるツールだ。
今使っているAntigravity（このチャット画面）と同じエンジンを、ターミナルだけで動かせる。

デスクトップ版と何が違うか。一言で言えば、**「AIがコマンドをそのまま実行できる」** 点だ。
「調べて→コードを書いて→実行する」という流れが、ターミナルの中で完結する。

### インストール

```bash
curl -fsSL https://antigravity.google/cli/install.sh | bash
source ~/.zshrc
agy --version
# → 1.0.2
```

インストール自体は1コマンド。PATHを通してバージョンが出ればセットアップ完了だ。

初回起動時はGoogleアカウントでのログインを求められる。ブラウザが自動で開く。

### 使い方

```bash
agy
```

起動後はターミナルに話しかけるだけ。

```
> このフォルダのファイルを一覧にして
> README.mdを要約して
```

今日試して面白かったのは、「Gitの最近のコミット履歴を少年漫画の必殺技風に大絶賛して」というプロンプトだ。
AIが勝手にGit履歴を読み込み、作業内容を「奥義」として解釈し始めた。
コードを実行できる環境にAIがいるというのは、デスクトップ版とは別の種類の感覚がある。

---

## セッション②：Twitter API v2 で自動投稿スクリプトを作る

### 構成

Python と `tweepy` ライブラリを使う。認証情報はコードに直書きせず、`.env` ファイルで管理する。

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

`.gitignore` に `.env` を含めることは必須だ。認証情報をGitに乗せることだけは絶対にやってはいけない。

### ハマりポイント

今日実際に詰まった箇所を正直に書いておく。

**403エラー（Forbidden）**
App permissionsが「Read only」のまま投稿しようとすると出る。
Developer Portal でアプリの権限を「Read and Write」に変更する必要があるが、**変更後にAccess Tokenを再発行しないと反映されない。** これを見落として30分くらい溶かした。

**402エラー（Payment Required）**
APIキーを正しく設定し、権限も直したのにこのエラーが出た。
原因はTwitterのプランが「Pay Per Use（残高ゼロ）」になっていたこと。
Developer Portal でFreeプランに切り替える必要がある。今日はここで時間切れになった。

---

技術的にはどちらも難しいものではない。ただ、セットアップの手順が細かく、どこかひとつ見落とすと詰まる。
今日は「動いた」というところまで至らなかったが、詰まった箇所は全部記録した。
次回はFreeプランへの切り替えから再開する。
