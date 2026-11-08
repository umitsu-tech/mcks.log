---
title: "【AgentCore】時刻を決めてハンズオン環境をお掃除する仕組みを用意してみた"
description: "社内勉強会向けに、Amazon Bedrock AgentCore のハンズオン環境を Terraform で作っています。"
pubDate: 2026-10-31
tags: ['AWS', 'AgentCore', 'Terraform', 'EventBridge', 'IAM']
qiitaId: b28dca1867058156de46
importedDate: 2026-11-08
qiitaStats:
  views: 1541
  likes: 2
  stocks: 0
  fetchedAt: 2026-11-08
---

## はじめに

社内勉強会向けに、Amazon Bedrock AgentCore のハンズオン環境を Terraform で作っています。

ベタではありますが、受講者1人に1台の EC2 を人数分立てて、ALB で受講者ごとに振り分けています。EC2 では code-server が動いていて、受講者はブラウザの中の VS Code から `agentcore deploy` を打ち、自分のエージェントを AgentCore Runtime に載せます。全体の構成は以前の記事にまとめています。

https://qiita.com/ryu-ki/items/222e0318ddb84b11fb82

環境を作ったはいいものの、いい感じに片付ける仕組みも合わせて用意したいものです。

EC2 と ALB は動いている限り課金されますし、受講者が `agentcore deploy` で作るものは Terraform の外にあるので、`terraform destroy` では消えません。

この記事では、終了時刻を1つ決めておくと自動で止まり、そのあと人が一覧を見てから消す、という後片付けの仕組みを用意してみたのでまとめます。

## 動作環境

| 項目 | バージョン・設定 |
|---|---|
| Terraform | 1.16.0 |
| AWS プロバイダー | 6.63.0 |
| AgentCore CLI | 0.28.1 |
| リージョン | 東京（ap-northeast-1） |

:::note
本記事は2026年10月時点の内容です。AgentCore と AgentCore CLI は変化が速いので、読む時点で変わっている可能性があります。
:::

## 今回やりたいこと

後片付けを「止める」と「消す」の2つに分けます。

![後片付けの流れ](https://images.ryu-ki-learn.com/agentcore-handson-env-cleanup/cleanup-flow.png)

止めるのは終了時刻で自動、消すのは翌日以降に主催者が一覧を見てから流す、という分担です。

基本的に止めれば課金されないリソースが大半なので、これで十分かと思います（EBS とスナップショットは止めても課金が続きますが、受講者12人分の EBS でも1日0.57ドルほどです）。消すほうは戻せないので、何を消すかを一覧で見てから人が決めたい、という理由もあります。

ただし ALB だけは例外で、停止することができないので、終了時刻に自動で削除することにしました。

また、受講者にも当日の最後に自分のデプロイを消してもらう形をとっています。受講者が自分で消し、消し忘れた分は翌日に主催者が拾う、という二段構えです。（仕組み化しているなら受講者はほったらかしでいいやんという説もあります）

## 手順

Terraform の変数で終了時刻を1つ渡すと、受講者ロールの Deny、EC2 の全台停止、ALB の削除の3つがまとめて仕込まれます。ここからは、その中身と、受講者が作ったものの消し方を順に書いていきます。

### 1. 終了時刻の渡し方

終了時刻は `handson_end_time` という変数で、`2026-10-29T21:15:00+09:00` のように秒とオフセットまで書いて渡します。書かなければこの仕組みごと作られないので、検証のときは外しておけます。

この時刻は2つの形で使います。

| 使う場所 | 必要な形 |
|---|---|
| EventBridge Scheduler の `at()` 式 | オフセットなしの日本時間とタイムゾーン |
| IAM の `DateGreaterThan` | UTC の時刻 |

ここで注意が必要なのが、Terraform の `formatdate` と `timeadd` はオフセットを UTC に直してくれないことです。

```bash
echo 'timeadd("2026-10-29T21:15:00+09:00", "0h")' | terraform console
# 実行結果
"2026-10-29T21:15:00+09:00"
```

`+09:00` のまま返ってくるので、このまま `formatdate` で末尾に `Z` を付けると、9時間ずれた UTC になってしまいます。

そこで、オフセットの部分を文字列で取り出し、その分を自分で引いてから UTC にしています。

```hcl
locals {
  deadline_enabled = var.handson_end_time != null

  # オフセットなしの日本時間（2026-10-29T21:15:00）は Scheduler に、UTC（2026-10-29T12:15:00Z）は IAM に渡す
  deadline_wall   = local.deadline_enabled ? substr(var.handson_end_time, 0, 19) : null
  deadline_offset = local.deadline_enabled ? substr(var.handson_end_time, 19, -1) : null
  deadline_shift = local.deadline_enabled ? (
    local.deadline_offset == "Z" ? "0h" :
    format("%s%sh%sm",
      substr(local.deadline_offset, 0, 1) == "+" ? "-" : "",
      substr(local.deadline_offset, 1, 2),
      substr(local.deadline_offset, 4, 2),
    )
  ) : null
  deadline_utc = local.deadline_enabled ? formatdate("YYYY-MM-DD'T'hh:mm:ss'Z'", timeadd("${local.deadline_wall}Z", local.deadline_shift)) : null
}
```

オフセットを外した日本時間の後ろに一度 `Z` を付けて UTC のふりをさせ、`+09:00` なら `-09h00m` を足す、という流れです。

apply のあとは、出力 `handson_deadline` に日本時間と UTC の両方を出すようにしていて、意図した時刻になっているかをそこで確かめます。

https://developer.hashicorp.com/terraform/language/functions/timeadd

https://developer.hashicorp.com/terraform/language/functions/formatdate

### 2. 受講者ロールの時刻つき Deny

受講者の EC2 に付けているロールに、時刻を条件にした Deny を足します。

```hcl
resource "aws_iam_role_policy" "deadline" {
  count = local.deadline_enabled ? 1 : 0
  name  = "${local.name}-deadline"
  role  = aws_iam_role.instance.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "DenyModelCallsAfterEnd"
      Effect = "Deny"
      Action = [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream",
        "bedrock-agentcore:InvokeAgentRuntime",
      ]
      Resource = "*"
      Condition = {
        DateGreaterThan = { "aws:CurrentTime" = local.deadline_utc }
      }
    }]
  })
}
```

`aws:CurrentTime` はリクエストの時刻を表す条件キーで、`DateGreaterThan` と組み合わせると「この時刻より後なら拒否」になります。ポリシーを1つ足すだけで、追加のリソースは要りません。

止めているのは、Bedrock のモデル呼び出しと、Runtime の呼び出し（`InvokeAgentRuntime`）です。Runtime の中からのモデル呼び出しは Runtime の実行ロールで動くので、受講者ロールの Deny は届きません。そのかわり、受講者の EC2 から Runtime を呼ぶところを止めています。

終了時刻になると EC2 も同時に止まるので、Deny が効いているかは、止まった EC2 を1台だけ起こしてモデルを呼んで確かめました。`AccessDeniedException`（explicit deny）が返ってきて、きちんと拒否されていました。

https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-currenttime

https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html#Conditions_Date

### 3. Scheduler による EC2 の停止と ALB の削除

EC2 の停止と ALB の削除は、EventBridge Scheduler に1回だけ動くスケジュールを登録して行います。

```hcl
resource "aws_scheduler_schedule" "stop_instances" {
  count = local.deadline_enabled ? 1 : 0
  name  = "${local.name}-stop-instances"

  schedule_expression          = "at(${local.deadline_wall})"
  schedule_expression_timezone = var.handson_timezone
  action_after_completion      = "DELETE"

  flexible_time_window {
    mode = "OFF"
  }

  target {
    arn      = "arn:${data.aws_partition.current.partition}:scheduler:::aws-sdk:ec2:stopInstances"
    role_arn = aws_iam_role.scheduler[0].arn
    input = jsonencode({
      InstanceIds = [for i in aws_instance.participant : i.id]
    })
  }
}
```

ポイントは3つあります。

まず、`at()` 式にはオフセットを外した日本時間（`2026-10-29T21:15:00`）を渡し、タイムゾーンは `schedule_expression_timezone`（既定は `Asia/Tokyo`）で指定します。

次に、ターゲットはユニバーサルターゲット（`aws-sdk:<サービス>:<API>`）にしています。AWS の API を直接呼べるので、`ec2:stopInstances` を呼ぶためだけの Lambda を置かずに済みます。

最後に、`action_after_completion = "DELETE"` で、実行が終わったスケジュールを自分で消すようにしています。一回きりのスケジュールは実行後もアカウントのクォータに数えられるので、消しておくことがドキュメントでも勧められています。

ALB の削除も同じ形で、ターゲットを `aws-sdk:elasticloadbalancingv2:deleteLoadBalancer` にしています。ここで少しややこしいのが、Scheduler のロールに許可する IAM のアクションは `elasticloadbalancing:DeleteLoadBalancer` で、名前が一致しないことです。ユニバーサルターゲットの ARN には AWS SDK のサービス名を書くことになっていて、IAM のアクションの接頭辞とは違うことがある、とドキュメントに書かれています。

Scheduler のロールは、この環境の EC2 と ALB の ARN だけを対象にしています。

また、Scheduler は60秒の精度で動くので、例えば21:15に設定すると21:15:00〜21:15:59のどこかで実行されます。

12人分で動かしたときは、21:15の設定で12台とも21:15:21に止まり、ALB と2つのスケジュールも消えていました。

https://docs.aws.amazon.com/scheduler/latest/UserGuide/schedule-types.html#one-time

https://docs.aws.amazon.com/scheduler/latest/UserGuide/managing-targets-universal.html

### 4. 終了時刻を過ぎたあとの注意点

ALB は Terraform の外で消えるので、時刻を過ぎたあとに `terraform apply` を打つと、ALB が作り直されてしまいます。

そのため、時刻を過ぎたら apply は打たず、`terraform destroy` だけにしています。destroy は、すでに消えているものを refresh で検知してくれるので、そのまま通ります。

終了時刻を延ばしたり早めたりしたいときは、時刻が来る前に `handson_end_time` を書き換えて apply します。変わるのは Deny と2つのスケジュールだけで、EC2 は作り直されません。

:::note warn
開催が終わったら、設定から `handson_end_time` を外しておきます。過ぎた時刻を書いたまま次に apply すると、Deny が最初から効いた状態になってしまうためです。
:::

### 5. 受講者が作ったものの棚卸し

受講者が `agentcore deploy` で作るものは、Terraform の外に次のように残ります。

| もの | 名前 | 消え方 |
|---|---|---|
| CloudFormation スタック | `AgentCore-handsonuserNN-default` | 削除が必要 |
| AgentCore Runtime | `handsonuserNN_agent` | スタックと一緒に削除 |
| IAM ロール（実行ロール） | `AgentCore-handson` で始まる名前 | スタックと一緒に削除 |
| CloudWatch ロググループ | `/aws/bedrock-agentcore/runtimes/handsonuserNN_agent-...` | スタック削除後も残存 |

デプロイは CodeZip 方式なので、受講者ごとの ECR リポジトリは作られません。

https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-cli-reference.html

これらを、名前の接頭辞で選んで消すスクリプトを用意しました。

```bash
scripts/cleanup-participant-resources.sh            # 一覧だけ。何も消さない
scripts/cleanup-participant-resources.sh --delete   # 一覧のあと削除する
```

対象は名前が `handson` で始まるものだけです。

名前で選べるのは、受講者に配っているエージェントの雛形で `agentcore.json` のプロジェクト名を `handsonPARTICIPANT` にしておき、EC2 の起動時に `handsonuser01` のように受講者ごとの名前へ置き換えているためです。Runtime もスタックもロールもロググループも、この名前から同じ規則で決まります。

消すときは、次の順番で進みます。

![棚卸しスクリプトが消すもの](https://images.ryu-ki-learn.com/agentcore-handson-env-cleanup/cleanup-script-targets.png)

1. CloudFormation のスタックを消す（Runtime と実行ロールはスタックの中にあるので、ここで一緒に消える）
2. スタックを消したあとも Runtime やロールが残っていれば、それを直接消す
3. ロググループを消す（スタックを消しても残るため）

CDK のブートストラップ（`cdk bootstrap` で作られるもの）は次回も使うので触りません。

https://github.com/umitsu-tech/agentcore-handson-env/blob/main/scripts/cleanup-participant-resources.sh

後片付けは、棚卸しスクリプトと `terraform destroy` の2つで終わります。消す範囲は次のように分かれています。

| 道具 | 消すもの |
|---|---|
| 棚卸しスクリプト | 受講者が `agentcore deploy` で作ったもの |
| `terraform destroy` | EC2・ALB・VPC など Terraform で作ったもの |

流す順番は「棚卸し → destroy」にしています。

12人分を片付けたあとに一覧モードでもう一度流すと、4種類とも残っていませんでした。

```bash
scripts/cleanup-participant-resources.sh
# 実行結果

==> CloudFormation スタック（AgentCore-handson*）
(なし)

==> AgentCore Runtime（handson*）
(なし)

==> IAM ロール（AgentCore-handson*）
(なし)

==> CloudWatch ロググループ（/aws/bedrock-agentcore/runtimes/handson*）
(なし)

一覧だけ表示しました。削除するときは --delete を付けて実行してください
```

## おわりに

以上、時刻を決めてハンズオン環境をお掃除する仕組みについて書いてきました。

せっかく開催して盛況に終わったとしても、リソースの消し忘れで余計な出費があるとかなしいので気をつけたいですね。

同じようなハンズオン環境を構築したい方の参考になれば幸いです。

ありがとうございました。
