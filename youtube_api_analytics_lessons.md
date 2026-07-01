---
title: "投稿直後の動画も追いたい！YouTube Analytics APIの「24時間遅延問題」をData APIとの併用で解決した話"
emoji: "📊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["youtube", "python", "api", "analytics", "個人開発"]
published: false
---

TikTokからYouTube Shortsへのマルチプラットフォーム展開を行うにあたり、動画のパフォーマンス（再生数、いいね数、視聴割合など）を自動収集するPythonスクリプトを実装しました。

その中で、**YouTube Analytics API** 特有の「データ反映の遅延問題」に直面し、それを **YouTube Data API v3** を併用することで対処したので、その学びと実装のポイントをまとめます。

## 直面した課題：YouTube Analytics API の遅延問題

YouTubeのチャンネル運用において、「投稿直後の初速」は非常に重要です。そこで、以下のようなコードを書いて Analytics API (`youtubeAnalytics.reports().query`) からデータを取得しようとしました。

```python
# Analytics API で直近の動画データを取得しようとするも...
request = youtube_analytics.reports().query(
    ids='channel==MINE',
    startDate=start_date,
    endDate=today,
    metrics='views,likes,comments,shares,averageViewDuration,averageViewPercentage',
    dimensions='video',
    maxResults=200,
    sort='-views'
)
```

しかし、何度実行しても **昨日や今日投稿したばかりの動画のデータが空（0件）** として返ってきてしまいます。

原因を調査した結果、YouTube Analytics API にはGoogle側の集計処理の都合上 **24〜48時間（約1〜2日）のタイムラグがある** ことが判明しました。
つまり、「昨日投稿した動画の視聴割合（Viewed Percentage）が知りたい」と思っても、翌日や翌々日にならないとAPIからデータが取得できないのです。

## 解決策：YouTube Data API v3 との併用

投稿直後の「今の再生回数」や「今のいいね数」は、YouTube Studioアプリ上ではリアルタイムで確認できます。

このリアルタイムな基本データを提供しているのが、**YouTube Data API v3** です。
Data API の `videos().list` エンドポイントを使うと、遅延なしで現在の `viewCount` や `likeCount` が取得できます。ただし、Analytics APIで取得できるような詳細な指標（`averageViewPercentage` 等）は取得できません。

そこで、これら2つのAPIを併用し、不足しているデータを補完する処理を実装しました。

### 実装フロー

1. **チャンネルの動画一覧を取得**
   Data API (`channels().list` & `playlistItems().list`) を使って、アカウントにアップロードされている全動画のIDリストを取得します。これで新しい動画も漏れなく把握できます。
2. **Data APIでリアルタイム統計を取得**
   動画IDのリストを元に Data API (`videos().list`) を実行し、最新の再生回数・いいね数・コメント数を取得します。
3. **Analytics APIで詳細分析データを取得**
   同じ期間に対して Analytics API を実行し、集計済みの分析データを取得します。
4. **データのマージ**
   - Analytics API にデータが存在する場合：その値（詳細指標を含む）を採用する。
   - Analytics API にデータがまだ無い場合（直近の動画）：Data API のリアルタイム値（再生回数、いいね、コメント）で補完し、Analytics固有の指標（閲覧割合など）は `N/A` として扱う。

### 実装のメリット

この構成により、以下のような出力（CSV等）が得られるようになります。

| Video Title | Views | Likes | Viewed % | Data Source |
| :--- | :---: | :---: | :---: | :---: |
| 【古い動画】猫の日常 | 120,400 | 2,100 | 78.5 | Analytics API (Delayed) |
| 【最新動画】本日の猫 | 1,513 | 15 | N/A | Data API (Realtime) |

古い動画は詳細な分析データが取得でき、投稿直後の動画であっても空にならず「今の再生回数」がリアルタイムで追える自動収集スクリプトが作成できました。

## その他のハマりポイント・学び

### 1. GCPのOAuth設定画面のUI変更
2024〜2026年にかけてGoogle Cloud ConsoleのUIが「Google Auth Platform」へと刷新されており、古い記事にある「OAuth同意画面」の設定場所が変わっていました。
テストユーザー（自分のアカウント）を追加するには、新しく用意された **「対象 (Audience)」タブ** からメールアドレスを登録する必要があります。これを忘れると `Error 403: access_denied` でログインできずエラーになります。

### 2. セキュリティ：トークンファイルの除外漏れに注意
OAuth認証フローをローカルで完結させると、認証情報の `credentials.json` と、アクセス権が含まれた `token.json` が生成されます。
誤って `git add .` をしてしまうと、これらのシークレットがGitHubに漏洩してしまいます。**実装を始める前に必ず `.gitignore` に追加しておく** のが基本です。

```gitignore
# .gitignore
credentials.json
token.json
```

## まとめ

YouTube APIを使ってデータ分析を自動化する際、Analytics APIだけでは直近のデータが追えません。
しかし、Data APIと組み合わせることで「即時性」と「詳細な分析」を両立したデータ収集スクリプトが構築できます。

これからYouTubeのデータをPython等で分析したいと考えている方の参考になれば幸いです。
