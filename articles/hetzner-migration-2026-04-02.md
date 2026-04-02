---
title: "非エンジニアがClaude Codeで7サイトのインフラ移行をやりきった話"
emoji: "🦉"
type: "idea"
topics: ["claudecode", "インフラ", "wordpress", "cloudflare"]
published: false
---

## 自分はエンジニアじゃない

Webマーケのコンサルタントをやっている。ABテストの設計とか、GA4の計測設計とか、そういう仕事。コードは書けない。

2025年後半からClaude Codeを使い始めて半年くらいになる。

## やったこと

WordPressのサイトを7つ持っていて、全部「さくらレンタルサーバー」で動かしていた。これをHetzner VPS（CAX11、ARM、月€4.49）1台に全部引っ越した。

なんでかというと、Claude Codeと一緒に仕事するようになって、「Claude CodeがサーバーをSSHで直接いじれたら最強じゃない？」と思ったから。さくらは管理画面ポチポチのサービスで、Claude Codeからは触りにくい。それが嫌だった。

## 構成

引っ越した後の構成はこうなった。Claude Codeが全部作ってくれたもので、自分はこれを「こういう感じにしたい」と伝えただけ。

```
Hetzner VPS CAX11（€4.49/月）
├── Caddy（リバースプロキシ、SSL終端）
├── WordPress × 4サイト（Docker コンテナ）
├── MySQL 8.0（WP共有）
├── n8n（自作のSlack Bot用）
├── PostgreSQL（n8n用）
└── walkerOS collector（計測用）

Cloudflare（無料枠）
├── CF Pages × 2（oshii.net, dora3.jp の静的配信）
├── DNS
├── SSL
└── Email Routing
```

正直、この構成図の中身を全部理解しているかというと怪しい。CaddyとかwalkerOSとか、Claude Codeが「これを使いましょう」って提案してきて、自分は「OK」と言っただけ。

## 自分は何をしたか

コードは1行も書いていない。自分がやったのは：

- 「さくらからHetznerに引っ越す」と決めた
- 「oshii.netとdora3.jpは静的化してCF Pagesに載せよう」と決めた
- 「バックアップは日次DB + 週次uploads + Google Driveのオフサイト」と方針を決めた
- 移行のたびにブラウザで目視確認した

Claude Codeがやったのは：

- Docker Compose, Caddyfile, deploy.sh（wget --mirror → URL置換 → wrangler pages deploy）の作成
- Cloudflare APIでDNSレコード設定
- backup.shの作成とcron設定
- 全部のトラブルシュートと修正

## 具体的に何が起きたか

### CSS全壊事件

oshii.netを静的化したとき、Claude Codeがwget --mirrorでサイトを丸ごとダウンロードして、HTMLのURLを書き換えて、CF Pagesにデプロイした。一発で動いた…と思ったらCSSが全壊していた。

原因は`global-styles-inline-css`というWordPressが吐くインラインCSSを「不要」と判断して消したこと。実はこれにGutenbergテーマ（SANGO）のCSS変数・レイアウト・フォント設定が全部入っていた。戻したら直った。

自分はこれの何がまずかったのかを正確には理解していない。Claude Codeが「このCSSを消したのが原因です、戻しますね」と言って、戻して、直った。自分はブラウザで「あ、直った」と確認しただけ。

### DNS吹っ飛び事件

oshii.netのNSをさくらからCloudflareに切り替えたとき、サブドメイン（edit.oshii.net, test.oshii.net, o.oshii.net, t.oshii.net）のDNSレコードが全部消えた。NS変更するとそうなるらしい。知らなかった。

Claude Codeが「先にサブドメインのレコードをCloudflare側に作っておく必要がありました」と言って、全部設定し直してくれた。

### .jpドメイン移管できない問題

「ドメインも全部Cloudflare Registrarに寄せよう」と思って移管しようとしたら、.jpドメインは非対応だった。Claude Codeに「日本でAPI/CLIからドメイン管理できるレジストラを調べて」と頼んだら、バリュードメインとラッコドメインの2社しかないことがわかった。個人で1〜2ドメインならバリュードメイン一択。

## 数字

| | Before | After |
|---|---|---|
| サーバー費用 | さくらレンタル（月額忘れた） | Hetzner €4.49/月（約700円） |
| TTFB（oshii.net） | 1.5〜2秒 | 0.05〜0.08秒 |
| バックアップ | 手動（やってなかった） | 日次自動 + 週次オフサイト |
| Claude Codeからの操作 | ほぼ不可 | SSH/API経由で全操作可能 |
| 移行期間 | — | 約2週間（他の仕事の合間に） |

## 正直な感想

技術の中身はよくわかってない。Docker ComposeもCaddyfileもdeploy.shも、Claude Codeが書いたものをそのまま使っている。中身を読んで「なるほど」とはならない。

でも「何をやりたいか」は自分にしか決められない。「このサーバーに引っ越したい」「静的化して速くしたい」「バックアップは自動にしたい」という判断は自分がやって、実行はClaude Codeがやる。

逆にうまくいかないのは、自分が「何をやりたいか」をちゃんと言葉にできないとき。「なんかいい感じにして」だとClaude Codeも困る。マーケの仕事でクライアントに「で、どうしたいんですか？」って聞くのと同じで、やりたいことの解像度が低いとAIも動けない。

**「何をやるか決める」「やりたいことを言葉にする」は自分の仕事。「やる」のはAIの仕事。** この分業がしっくりきている。
