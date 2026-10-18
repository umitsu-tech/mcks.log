---
title: "【DuckDB】ブログの記事検索を DuckDB Wasm + Parquet で作ってみた"
description: "先日、Qiita に投稿してきた記事を自分のブログ（Astro + Cloudflare Workers の静的サイト）へ一括移植したところ、記事数が 118 本になりました。一覧…"
pubDate: 2026-10-18
tags: ['Astro', 'CloudflareWorkers', 'DuckDB', 'Parquet', 'WebAssembly']
qiitaId: 607e1090aa78323a56c1
importedDate: 2026-10-18
qiitaStats:
  views: 18
  likes: 0
  stocks: 0
  fetchedAt: 2026-10-18
---

## はじめに

先日、Qiita に投稿してきた記事を自分のブログ（Astro + Cloudflare Workers の静的サイト）へ一括移植したところ、記事数が 118 本になりました。一覧をページ分けして見やすくはしたものの、「あの記事どこだっけ」を探す手段がありません。

https://blog.ryu-ki-learn.com/

静的サイトの検索といえば Pagefind などの定番ツールがあるそうですが、どうせなら使ってみたい技術がありました。DuckDB Wasm です。ブラウザの中で DuckDB がそのまま動き、SQL が打てるという代物です。今回のような小規模な情報の検索に使うには大げさ気味なのですが、使いたいので使います。

:::note
既製の検索エンジンに頼らず自分のサイトに合う検索を自作する、という発想はこちらの記事に大いに刺激を受けています。

https://voluntas.ghost.io/shiguredo-document-tool-mikan/

:::

## 動作環境

| 項目 | バージョン |
|------|-----------|
| @duckdb/duckdb-wasm | 1.32.0 |
| @duckdb/node-api | 1.5.4 |
| Astro | 7.0.6 |
| Node.js | 22 系（Workers Builds のビルド環境） |

:::note
本記事の内容は 2026 年 8 月時点のものです。
:::

## 全体構成

仕組みはシンプルで、「ビルド時に作る検索インデックス（Parquet）」と「ブラウザで動く DuckDB Wasm」の 2 つからなります。Parquet は、必要な列だけを読み出せる列指向のデータファイル形式です。

![検索の全体構成](https://images.ryu-ki-learn.com/duckdb-wasm-blog-search/overview.png)

CDN から静的ファイルを配るだけの構成で全文検索ができます。

![search-ui.png](https://images.ryu-ki-learn.com/duckdb-wasm-blog-search/search-ui.png)

## ビルド時のインデックス生成

インデックスの生成は Astro の integration として組み込みました。ビルドコマンドに手を入れる方法もありますが、私の環境は CI（Workers Builds）のビルドコマンドが `npx astro build` で固定されているため、こちらの方法にしました。

やっていることは、全記事の Markdown からタイトル・タグ・本文（Markdown 記法を落としたプレーンテキスト）を集めて、Node 版の DuckDB（`@duckdb/node-api`）で Parquet に書き出すだけです。

https://duckdb.org/docs/current/clients/node_neo/overview

```sql
COPY (SELECT * FROM read_json_auto('records.json') ORDER BY pubDate DESC)
TO 'search-index.parquet' (FORMAT parquet, COMPRESSION zstd)
```

ひと工夫したのは日本語の表記ゆれ対応です。たとえば記事に「AWS」と書いてあっても、検索窓に全角の「ＡＷＳ」を打つとヒットしません。大文字小文字（aws / AWS）も同様です。

こうした字形のゆれを標準形に揃える変換が NFKC 正規化と呼ばれるものですが、DuckDB にはこの関数がありません。そこで正規化は SQL ではなく JavaScript 側でやることにしました。

インデックスを作るときに `normalize('NFKC')` と `toLowerCase()` を通した検索用の列（`n_title`, `n_body` など）を持たせておき、検索するときも入力された語に同じ変換をかけます。書く側と探す側の両方を同じ形に揃えてから比べる、という分担です。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/String/normalize

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/String/toLowerCase

118 記事・本文込みで約 590KB になりました。

## ブラウザ側の検索処理

エンジン本体（DuckDB Wasm）は 30MB 超あるので、ページ表示時には読み込みません。検索窓にフォーカスした瞬間に動的 import で遅延ロードします。

検索クエリはこんな感じの SQL になります（実際は prepared statement でパラメータ化しています）。

```sql
SELECT slug, collection, title, pubDate, body
FROM read_parquet('https://blog.example.com/search-index.parquet')
WHERE (n_title LIKE '%語1%' OR n_tags LIKE '%語1%' OR n_body LIKE '%語1%')
  AND (n_title LIKE '%語2%' OR ...)
ORDER BY pubDate DESC
LIMIT 30
```

形態素解析も転置インデックスもない、シンプルな部分一致です。それでも 118 記事の総当たりなら数十ミリ秒で返ってきます。結果は新しい順に並べ、ヒットした語はタイトルと本文スニペットの両方でハイライトして、「なぜこの記事がヒットしたのか」が一目でわかるようにしました。

DuckDB Wasm はファイルの必要な部分だけを読むので、インデックスが増えても検索に必要な列だけを取りに行ってくれます。

## Workers の 25MB 制限

順調に見えた実装ですが、本番デプロイで止まりました。当初、wasm ファイルはサイトに同梱するセルフホスト方式にしていたのですが、Cloudflare Workers の静的アセットには 1 ファイル 25MB の上限があり、DuckDB の wasm（32.7MB と 37.5MB）が超えてしまっていました。ローカルのビルドとプレビューでは何の問題もなく動くため、デプロイして初めて発覚しました。

対処として、wasm 本体だけは公式配布の jsDelivr CDN からロードする方式に変えました。検索インデックス（記事データ）は自サイト配信のままなので、「データは手元、汎用エンジンだけ CDN」という分担です。なお CDN 上の worker を直接 `new Worker()` するとブラウザにブロックされるため、blob URL を経由する公式の回避策を使っています。

https://duckdb.org/docs/current/clients/wasm/instantiation.html

```js
const bundle = await duckdb.selectBundle(duckdb.getJsDelivrBundles());
const workerUrl = URL.createObjectURL(
  new Blob([`importScripts("${bundle.mainWorker}");`], { type: 'text/javascript' })
);
const worker = new Worker(workerUrl);
```

## DuckDB である必要はあったのか

正直に言うと、ありません。118 記事・1MB 程度のテキストであれば、インデックスを JSON で配って JavaScript の `includes` で総当たりしても数ミリ秒で同じ結果が得られます。エンジン 30MB のダウンロードも要りません。

それでも DuckDB を選んだ理由は、検索ロジックを SQL で書けること（人気順ソートやタグ絞り込みも SQL 一行で足せます）、Parquet の部分読みのおかげで記事が数千本になっても設計を変えずに済むこと、そして何より、ブラウザで SQL が動くという体験そのものが面白かったからです。目的が「検索機能の実装コスト最小化」なら他を選ぶべきで、「遊びと学びと将来性」なら DuckDB はいい選択だったと思い（込んでい）ます。

## おわりに

以上、DuckDB Wasm + Parquet で、サーバーレスなブログ全文検索を作りました。シンプルな LIKE 検索ですが、自分のブログを自分で探す用途には十分機能しています。

初回ロードの重さ（実測で 10 秒ほど）は今後の課題なので、追々改善できればいいなと思っています。

ありがとうございました。
