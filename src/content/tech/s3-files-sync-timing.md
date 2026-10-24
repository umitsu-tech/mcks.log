---
title: "【S3 Files】ファイルシステムと S3 バケットの同期にかかる時間を測ってみた"
description: "2026年4月7日に、Amazon S3 Files が登場しました。"
pubDate: 2026-10-24
tags: ['AWS', 'EC2', 'nfs', 'S3', 'S3files']
qiitaId: ac1152359bcf1217c1f5
importedDate: 2026-10-24
qiitaStats:
  views: 16
  likes: 0
  stocks: 0
  fetchedAt: 2026-10-24
---

## はじめに

2026年4月7日に、Amazon S3 Files が登場しました。

https://aws.amazon.com/blogs/aws/launching-s3-files-making-s3-buckets-accessible-as-file-systems

S3 の汎用バケットを、ファイルシステムとしてマウントできるサービスです。EC2 からはふつうのファイルとして読み書きでき、裏では S3 バケットと同期してくれます。

便利そうだなと思う一方で、ふと気になったのが反映までの時間です。ファイルシステムに書いたものが S3 に出ていくまで、逆に S3 に置いたものがファイルシステムから見えるまで、それぞれどれくらいかかるのでしょうか。ドキュメントにもある程度書いてありますが、実際の待ち時間は動かしてみないとわかりません。

そこで、EC2 からマウントして、両方向の同期にかかる時間を測ってみました。

この記事では、その測り方と結果、待ち時間の内訳をまとめます。

:::note
この結果は先日の AWS Community Builders Meetup Kyushu の LT でも、博多ラーメンの麺の茹で加減になぞらえて話しました。

https://speakerdeck.com/ryuki0947/what-hakata-ramen-teaches-us-about-the-boil-time-of-star-star
:::

## S3 Files とは

S3 Files は、裏側に Amazon EFS の仕組みを使ったサービスです。EC2 のほか、ECS、EKS、Lambda からもマウントできます。

データの本体はあくまで S3 バケットにあり、ファイルシステム（以下 FS）と S3 バケットのあいだは裏で同期されます。FS から S3 へ書き出すのがエクスポート、S3 から FS へ取り込むのがインポートです。

![S3 Files の同期の全体像](https://images.ryu-ki-learn.com/s3-files-sync-timing/s3files-sync.png)

同期のタイミングについて、ユーザーガイドには次のように書かれています。

https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-synchronization.html

まず FS から S3 への向きです。

> When a file is modified on the file system, S3 Files waits for a period of write inactivity (60 seconds) before exporting those changes back to your S3 bucket.

訳すと、「FS でファイルが変更されると、S3 Files は書き込みのない期間（60秒）を待ってから S3 バケットへ書き出す」とのことです。同じファイルへの立て続けの書き込みは、1回の PUT にまとめられるとも書かれています。

逆に S3 から FS への向きは、S3 のイベント通知でバケットの変更を検知して反映する仕組みです。ただ、こちらは何秒で反映されるかまでは書かれていません。

公式ブログのほうは、FS から S3 は数分以内、S3 から FS は数秒以内（ときに1分以上）と、ざっくりした書き方です。

なお、イベント通知で反映されるのは、FS にデータが載っているファイルです。しばらく読まれずに FS から消えたファイルは、次に読んだときに S3 から最新の版が取り込まれます。今回測ったのは、新しく置いた小さなファイルです。

## 前提環境

| 項目 | 内容 |
|---|---|
| リージョン | 東京（ap-northeast-1） |
| EC2 | t4g.micro |
| OS | Amazon Linux 2023（arm64） |
| S3 Files のクライアント | amazon-efs-utils 3.3.3 |
| Python | 3.9（Amazon Linux 2023 の標準の python3） |
| boto3 | 1.40.31 |
| S3 バケット | バージョニング有効 |

:::note
本記事の内容は 2026年10月時点のものです。
:::

S3 Files を使うには、バケットのバージョニングを有効にしておく必要があります。EC2 には amazon-efs-utils を入れて、S3 Files のマウントヘルパーでマウントしました。

```bash
sudo mount -t s3files fs-xxxxxxxxxxxxxxxxx:/ /mnt/s3files
```

https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-mounting.html

## 今回やりたいこと

EC2 の上で Python のスクリプトを動かして、次の4つを測ります。

| 実験 | やること | 回数 |
|---|---|---|
| FS → S3 | FS に書いて S3 の HeadObject が通るまでの秒数 | 20回 |
| S3 → FS | S3 に PutObject して FS から見えるまでの秒数 | 20回 |
| 書き足し続ける | 5秒ごとの追記を115秒続けて S3 にバージョンが出る時刻 | 1回 |
| 両側から書く | FS に書いた10秒後に S3 へ同じキーで置いたときの FS の中身 | 1回 |

上の2つを図にすると以下のような感じです。

![測り方](https://images.ryu-ki-learn.com/s3-files-sync-timing/measure-setup.png)

FS → S3 は、5秒おきにファイルを書いて、それぞれ0.5秒ごとに S3 を確認しました。メインの部分だけ抜き出すとこんな感じです。

```python
def trial(i):
    key = f"{d}/file-{i:02d}.txt"
    write_file(f"{MNT}/{key}", f"file {i} written at {time.time()}\n")  # 書いて fsync する
    t0 = time.time()
    while time.time() - t0 < timeout:
        if head_exists(key):  # s3.head_object が通れば S3 に出てきた
            results[i] = round(time.time() - t0, 2)
            return
        time.sleep(0.5)
```

S3 → FS は逆に、3秒おきに S3 へ PutObject して、それぞれ0.2秒ごとに FS で `os.path.exists` を確かめました。

```python
def trial(i):
    key = f"{d}/object-{i:02d}.txt"
    s3.put_object(Bucket=BUCKET, Key=key, Body=f"object {i}\n".encode())
    t0 = time.time()
    while time.time() - t0 < timeout:
        if os.path.exists(f"{MNT}/{key}"):  # FS から見えたら終わり
            results[i] = round(time.time() - t0, 2)
            return
        time.sleep(0.2)
```

https://docs.aws.amazon.com/AmazonS3/latest/API/API_HeadObject.html

https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html

確認の間隔（0.5秒と0.2秒）のぶん、結果には誤差があります。

では、早速測ってみます。

## 同期の挙動

### FS → S3

20回とも61〜66秒で、中央値は62.6秒でした。

![FS → S3 は約1分](https://images.ryu-ki-learn.com/s3-files-sync-timing/chart-fs2s3.png)

ドキュメントの「書き込みが60秒止まってから」のとおりで、ほとんどぶれませんでした。60秒を数え終わってから S3 に PUT されるまでに、1〜6秒ほどかかっている計算です。

CloudWatch の `ExportAge`（FS の変更が S3 へ書き出されるのを待っている時間）も、実験中の最大は65.8秒でした。手元で測った値とほぼ同じです。

https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-files-monitoring-cloudwatch.html

### S3 → FS

こちらは、ふつうにマウントした状態だと3〜30秒でばらつき、中央値は16.5秒でした。

結果をよく見ると、少し不思議な並びになっていました。

```json
{"exp": "s3tofs", "seconds": [30.05, 27.05, 24.04, 21.04, 18.03, 15.03, 12.03, 9.02, 6.01, 3.01, 30.05, 27.05, 24.05, 21.04, 18.03, 15.03, 12.02, 9.02, 6.01, 3.01]}
```

3秒おきに置いたファイルの待ち時間が3秒ずつ短くなっていき、10個ごとに30秒へ戻っています。置いた時刻に関係なく、30秒ごとにまとめて見えるようになっていました。

最初は、S3 Files の取り込みが30秒ごとなのかと疑いました。ところが、NFS のキャッシュを切るオプション `noac` を付けてマウントし直して測ると、0.83〜8.41秒、中央値1.24秒まで縮みました。8.41秒は最初の1回だけで、理由までは追えていません。

```bash
sudo mount -t s3files -o noac fs-xxxxxxxxxxxxxxxxx:/ /mnt/noac
```

![S3 → FS は約1秒＋キャッシュ](https://images.ryu-ki-learn.com/s3-files-sync-timing/chart-s3tofs.png)

S3 Files 自体は、イベント通知を受けて約1秒で取り込んでいたようです。残りの最大30秒は、EC2 側で待たされていた時間でした。

#### 30秒の正体

S3 Files は NFS（v4.2）でマウントします。EC2 の OS が NFS クライアントになって、ネットワーク越しに S3 Files へ問い合わせる形です。間には、TLS で暗号化するための efs-proxy という中継が入ります。

![NFS の経路とキャッシュ](https://images.ryu-ki-learn.com/s3-files-sync-timing/nfs-path.png)

NFS クライアントは、毎回ネットワーク越しに問い合わせると遅いので、ファイルやディレクトリの情報をしばらくキャッシュします。Linux の既定値は次のとおりです。

| オプション | 対象 | 既定値 |
|---|---|---|
| `acregmin` / `acregmax` | ファイルの属性 | 3秒 / 60秒 |
| `acdirmin` / `acdirmax` | ディレクトリの情報 | 30秒 / 60秒 |

https://man7.org/linux/man-pages/man5/nfs.5.html

`os.path.exists` で新しいファイルを探すと、「そのファイルは無い」という結果も、ディレクトリのキャッシュが切れるまで覚えられてしまいます。S3 Files 側ではもう取り込み済みなのに、EC2 側では30秒（`acdirmin`）のあいだ「無い」が返り続けていた、ということのようです。

今回は3秒ごとにファイルが増えていたので毎回30秒で区切られていましたが、ディレクトリに変化の無い時間が続くと、キャッシュの時間は `acdirmax` の60秒まで延びていきます。ファイルがたまにしか増えない使い方だと、最大60秒待つこともありそうです。

![S3 → FS の待ち時間の内訳](https://images.ryu-ki-learn.com/s3-files-sync-timing/s3tofs-breakdown.png)

S3 Files のマウントヘルパーは、NFS のバージョンや通信まわりのオプションを自動で付けてくれますが、キャッシュの秒数（`ac` から始まるオプション）は指定していません。そのため、Linux の既定値がそのまま影響していそうです。測ったときの `/proc/mounts` を見ても、`acdir` 系の指定はありませんでした。

:::note warn
`noac` にすると早く見えるようになりますが、毎回 S3 Files に問い合わせるうえ、書き込みも同期になるので、ふだんの読み書きは遅くなります。「無い」という結果だけをキャッシュしない `lookupcache=positive` や、`actimeo=10` のように秒数だけ短くする手もあるようですが、今回は試していません。どちらも上の nfs(5) のマニュアルに説明があります。
:::

### 書き足し続けたとき

5秒ごとに同じファイルへ追記を続けると、そのあいだは S3 に何も出てきませんでした。

![書き足し続けると S3 に出ていかない](https://images.ryu-ki-learn.com/s3-files-sync-timing/chart-stir.png)

115秒のあいだに24回書きましたが、S3 にバージョンが現れたのは181秒の時点の1回だけです。最後の書き込み（115秒）から65秒後で、FS → S3 と同じく60秒＋αでした。

ドキュメントにも、30秒ごとに5分間追記するアプリならエクスポートが始まるのは6分目、という同じ動きの例が載っていました。

ログのように書き足し続けるファイルは、書き足しが止まるまで S3 に出ていきません。S3 側でログを拾う仕組みを作るときは、気をつけたいところです。

### 両側から書いたとき

最後に、FS にファイルを書いて、まだ S3 に出ていかないうち（10秒後）に、S3 へ同じキーで別の内容を PutObject してみました。

![両側から書くと S3 が優先される](https://images.ryu-ki-learn.com/s3-files-sync-timing/conflict-flow.png)

12秒の時点で、FS の中身は S3 で置いた内容に置き換わりました。FS で書いた内容はファイルシステムの直下の `.s3files-lost+found-<ファイルシステム ID>/` に退避されていて、S3 には出ていきませんでした。

```json
{"exp": "conflict", "fs_timeline": [[10.08, "<FS で書いた中身>"], [12.14, "<S3 で置いた中身>"]], "lost_found": [["/.s3files-lost+found-<ファイルシステム ID>/d8009ae57fb721b1_<ファイル名>", "<FS で書いた中身>"]], "s3_versions": [["2026-10-16T16:45:13+00:00", "<S3 で置いた中身>"]]}
```

ドキュメントでも、衝突したときは S3 バケット側を優先すると書かれていて、そのとおりの動きでした。なお、lost+found に退避されたファイルは、消すまでファイルシステムのストレージ料金がかかるそうなので、こちらも忘れずに消しておきたいところです。

## 同期の待ち時間早見表

| 場面 | 待ち時間 | 主な理由 |
|---|---|---|
| FS → S3 | 約1分 | 書き込みが60秒止まってから書き出す |
| S3 → FS（S3 Files の取り込み） | 約1秒 | S3 のイベント通知 |
| S3 → FS（EC2 から見えるまで） | 今回は最大30秒（条件によって60秒） | NFS クライアントのキャッシュ |
| 書き足し続けるファイル | 止めてから約1分 | 60秒の数え直し |
| 両側から書いたとき | 約2秒で S3 の内容に | S3 側が優先 |

FS に書いてすぐ S3 側で使うなら、60秒＋αは待つつもりで組んだほうがよさそうです。S3 に置いたものを EC2 からすぐ見たいときは、NFS のキャッシュが効いていることを頭に入れておくと、ハマらずに済みそうです。待ち具合は、CloudWatch の `ExportAge` と `ImportAge` で見られます。

:::note
検証に使った EC2、S3 Files のファイルシステム、S3 バケットは、測り終わったあとに削除しています。
:::

## おわりに

以上、S3 Files の同期にかかる時間を測ってみました。

今回は EC2 からしか測っていないので、Lambda や ECS からマウントしたときにも同じキャッシュが効くのかは、またの機会に確かめてみたいです。

同じような疑問を持った方の参考になれば幸いです。

ありがとうございました。
