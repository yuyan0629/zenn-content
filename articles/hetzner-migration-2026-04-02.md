---
title: "非エンジニアがClaude Codeで7サイトのインフラ移行をやりきった話"
emoji: "🦉"
type: "idea"
topics: ["claudecode", "インフラ", "wordpress", "cloudflare"]
published: false
---

## 今日やったこと

さくらレンタルサーバーで動いていた7サイトを、Hetzner VPS 1台（€4.49/月）に全部移行した。Claude Codeと一緒に。

自分はWebマーケティングのコンサルタントで、エンジニアではない。

## Before / After

**Before:**
- さくらレンタルサーバーに7サイトが同居
- SSH/APIでの操作が制限的
- AI（Claude Code）から操作しにくい

**After:**
- Hetzner VPS（ARM €4.49/月）に Docker で全サイト集約
- 本番サイト（dora3.jp, oshii.net）は CF Pages で静的配信 → TTFB 1.5秒 → 0.07秒（20倍改善）
- 編集用WPはVPSのDockerコンテナ
- バックアップ：日次DB + 週次uploads + オフサイト（Google Drive）
- 全部CLIで操作できる → Claude Codeが直接触れる

```
Hetzner VPS（€4.49/月）
├── Caddy（リバースプロキシ）
├── WP × 4サイト（Docker）
├── n8n（AI秘書Bot）
├── PostgreSQL（n8n用）
├── MySQL（WP共有）
└── walkerOS（計測）

Cloudflare
├── CF Pages × 2（静的本番）
├── DNS管理
├── SSL終端
└── Email Routing
```

## 何が大変だったか

### WP静的化の罠

WordPressをwget --mirrorで静的化してCF Pagesに載せる、というシンプルな構成。だけど罠だらけだった。

- `global-styles-inline-css` を「不要」と判断して消したらサイト全壊。Gutenbergテーマの全CSSがここに入ってた
- URL一括置換でリンクテキストまで巻き込んで表示が崩れた
- NS変更したらサブドメインのDNSレコードが全部消えた
- コメント投稿がクロスドメインリダイレクトでブロックされた

全部Claude Codeと一緒にデバッグして解決したけど、1個ずつ踏んでいった感じ。

### .jpドメインがCloudflareで移管できない

「全部Cloudflareに寄せよう」と思ったら、.jpドメインは非対応だった。調べたら日本でAPI管理できるレジストラはバリュードメインとラッコドメインの2社のみ。

## Claude Codeとの分業

今回の移行で、Claude Codeがやったこと：

- Docker Compose の設計・作成
- Caddyfile の設定
- deploy.sh（wget → URL置換 → CF Pagesデプロイ）の作成
- DNS設定（Cloudflare API経由）
- バックアップスクリプト作成 + cron設定
- トラブルシュートの原因特定

自分がやったこと：

- 「さくらからHetznerに移行する」という意思決定
- 各ステップの方針判断（静的化する/しない、ドメインどうする等）
- 移行の優先順位
- 動作確認（ブラウザで見る）

**コードは1行も書いていない。** 方針を決めて、Claude Codeに指示して、結果を確認する。これの繰り返し。

## 学び

- 非エンジニアでもインフラ移行ができる時代になった（ただし「何をやりたいか」を言語化する力は要る）
- WP静的化は思ったより罠が多い。でもSSG書き直しよりは圧倒的に楽
- €4.49/月のVPS 1台で7サイト + AI秘書Bot + 計測基盤が全部動く。さくらより安い
- Cloudflareの無料枠が強すぎる（Pages, DNS, SSL, Email Routing全部無料）
