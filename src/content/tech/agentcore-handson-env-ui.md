---
title: "【AgentCore】エージェントのハンズオンをブラウザの中だけで完結させる教材を作った"
description: "社内勉強会向けに、Amazon Bedrock AgentCore のハンズオン環境を Terraform で作りました。"
pubDate: 2026-11-08
tags: ['AWS', 'AgentCore', 'StrandsAgents', 'Bedrock']
qiitaId: a6cb026b2d3a54d175cc
importedDate: 2026-11-08
qiitaStats:
  views: 72
  likes: 3
  stocks: 0
  fetchedAt: 2026-11-08
---

## はじめに

社内勉強会向けに、Amazon Bedrock AgentCore のハンズオン環境を Terraform で作りました。

受講者1人に1台の EC2 を用意し、EC2 で動く code-server をブラウザから開いてもらう構成です。全体の構成は以前の記事にまとめています。

https://qiita.com/ryu-ki/items/222e0318ddb84b11fb82

受講者には AWS の認証情報を配らず、URL とパスワードだけで参加してもらう方針にしています。

この記事では、ハンズオンの教材をブラウザの中だけで完結させるために考えたことをまとめます。

## 動作環境

| 項目 | バージョン・設定 |
|---|---|
| code-server | 4.141.0 |
| Strands Agents | 1.54.0 |
| bedrock-agentcore（Python SDK） | 1.22.0 |
| AgentCore CLI | 0.28.1 |
| モデル | Amazon Nova 2 Lite |
| リージョン | 東京（ap-northeast-1） |

:::note
本記事は2026年10月時点の内容です。AgentCore と AgentCore CLI は変化が速いので、読む時点で変わっている可能性があります。
:::

## ハンズオンのねらいと流れ

対象は初学者で、エージェントがループで動いていることと、AgentCore の Runtime や Memory が何を引き受けてくれるのかを知ってもらうのがねらいです。流れは次のとおりです。

1. 手元でエージェントを動かす
2. ツールを1つ足す
3. 会話を続けてみる
4. 自分のエージェントを Runtime に載せる
5. 振り返る

## ブラウザの中でやること

受講者がやることと、それをどこでやるかを並べると次のようになります。

| やること | やる場所 |
|---|---|
| コードを読む・ツールを足す | code-server のエディタ |
| 手元で動かす | ハンズオンの画面 |
| Runtime に載せる | code-server のターミナル |
| Runtime を呼ぶ | ハンズオンの画面 |
| 動きを見る | ハンズオンの画面 |
| 自分の Runtime を消す | code-server のターミナル |

ツールを足すところは、足す関数をあらかじめコメントで書いておき、受講者はコメントを外して保存するだけにしました。

## ハンズオンの画面

エージェントの中で何が起きているかは、ハンズオン用に用意した画面で見てもらうことにしました。1回のやりとりでループがどう回ったかを見せる画面です。

![ハンズオンの画面](https://images.ryu-ki-learn.com/agentcore-handson-env-ui/handson-ui-screen.png)

チャットの画面ではなく、エージェントの動作の様子がわかるように、Strands Agents が返すイベントを抽出して表示するようにしてみました。また、元のイベントは「生データを見る」の折りたたみの中に置きました。見たい人だけが開ける形です。

この画面は EC2 の中の3000番で動かしています。code-server には、同じマシンの別のポートで動いている画面を `/proxy/<ポート番号>/` の URL で開ける、組み込みのプロキシがあります。受講者は、code-server を開いている URL の後ろに `/proxy/3000/` を付けるだけでこの画面を開けます。

例えば受講者01なら次のとおりです。

| 開くもの | URL |
|---|---|
| code-server | `https://user01.handson.example.com/` |
| ハンズオンの画面 | `https://user01.handson.example.com/proxy/3000/` |

プロキシを通るアクセスにも code-server と同じ認証がかかるので、code-server にログインしていればそのまま開けます。3000番を ALB やセキュリティグループで外に開ける必要もありません。受講者が覚える URL もパスワードも増えない、という形です。

https://coder.com/docs/code-server/guide#accessing-web-services

## Runtime の呼び出しも同じ画面

後半で `agentcore deploy` を終えると、ハンズオンの画面に「手元で動かす / Runtime で動かす」の切り替えが現れます。Runtime 側に切り替えて質問すれば、Runtime に載せた自分のエージェントの動きが、前半と同じ形で流れます。

https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-cli-reference.html

![Runtime の呼び出しも同じ画面](https://images.ryu-ki-learn.com/agentcore-handson-env-ui/local-vs-runtime.png)

Runtime を呼んでいるのは画面のサーバーで、EC2 に付けた IAM ロールの権限で `InvokeAgentRuntime` を呼んでいます。どの Runtime を呼ぶかも `agentcore deploy` が書き出すファイルから読むので、受講者は ARN をコピーしたり認証情報を設定したりしなくて済みます。

https://docs.aws.amazon.com/bedrock-agentcore/latest/APIReference/API_InvokeAgentRuntime.html

## おわりに

以上、エージェントのハンズオンをブラウザの中だけで完結させるために考えたことを書いてきました。

ハンズオンの最後は、受講者が自分でデプロイした Runtime を、ターミナルで自分で消して終わります。消し忘れた分を主催者がまとめて片付ける仕組みは前の記事に書きました。

https://qiita.com/ryu-ki/items/b28dca1867058156de46

ただ振り返ると、少し過保護だったかもしれません。ツールはコメントを外すだけ、デプロイはコマンド1行、動きは用意した画面で見る、とつまずかないように整えたぶん、自分で書いて詰まって調べる経験はほとんど残せませんでした。最初に書いたねらいを、どこまで持ち帰ってもらえたかは、正直なところあまり自信がありません。

同じようなハンズオンを考えている方の参考になれば幸いです。

ありがとうございました。
