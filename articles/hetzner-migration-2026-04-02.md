---
title: "さくらレンタルサーバー → Hetzner VPS にWordPress 7サイトを移行した"
emoji: "🦉"
type: "idea"
topics: ["claudecode", "wordpress", "cloudflare", "hetzner"]
published: false
---

## 経緯

もともと別件でSlackで動くAIボット（n8n + Claude）を作っていて、そのためにHetzner VPS（CAX11、ARM、月€4.49）を契約していた。「さくらレンタルサーバーで管理してるWordPressのサイト7つも全部ここにぶちこめば管理楽じゃない？」と思って移行した。

さくらは管理画面ポチポチのサービスでClaude Codeから触りにくい。VPSならSSHで全部操作できる。

## 移行後の構成

```
Hetzner VPS CAX11（€4.49/月）
├── Caddy（リバースプロキシ、SSL終端）
├── WordPress × 4サイト（Docker コンテナ）
├── MySQL 8.0（WP共有）
├── n8n（Slack Bot用）
├── PostgreSQL（n8n用）
└── walkerOS collector（計測用）

Cloudflare（無料枠）
├── CF Pages × 2（静的配信）
├── DNS
├── SSL
└── Email Routing
```

7サイトのうち2サイトはwget --mirrorで静的化してCF Pagesで配信。残り4サイトはDocker上のWPをCaddy経由で配信。バックアップは日次DB + 週次uploads + 週次Google Drive。

## 踏んだ罠

### WP静的化でCSSが壊れる

`global-styles-inline-css`というWordPressが吐くインラインCSSを不要と判断して消したら、サイトの見た目が崩れた。GutenbergテーマのCSS変数・レイアウト・フォント設定が全部ここに入っている。消してはいけない。

### NS変更でサブドメインが消える

NSをさくらからCloudflareに切り替えたら、サブドメインのDNSレコードが全部消えた。旧NS側のレコードは引き継がれないので、先にCloudflare側にレコードを作っておく必要がある。

### .jpドメインがCloudflare Registrarに移管できない

.jpはCloudflare Registrar非対応。日本でAPI/CLIからドメイン管理できるレジストラはバリュードメインとラッコドメインの2社のみ。

## 数字

| | Before | After |
|---|---|---|
| サーバー費用 | さくらレンタル | Hetzner €4.49/月 |
| TTFB（静的化サイト） | 1.5〜2秒 | 0.05〜0.08秒 |
| バックアップ | なし | 日次自動 + 週次オフサイト |
| 移行期間 | — | 約2週間 |
