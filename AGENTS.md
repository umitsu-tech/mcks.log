# AGENTS.md

個人ブログ mcks.log のリポジトリ。Astro 7 + Cloudflare Workers（static assets）で https://blog.ryu-ki-learn.com として公開している。

## デプロイの仕組み

- main に push すると Workers Builds が自動で `npx astro build` → `npx wrangler deploy` を実行して本番反映される（手動デプロイ不要）
- カスタムドメイン・workers.dev無効化は wrangler.jsonc で宣言管理

## push の運用ルール（重要）

**mainへのpush = 本番デプロイ**。1回のpushで約2分のビルドが走る（無料枠は月3,000分）。

- 記事の投稿・修正のpushはいつでも自由に行ってよい
- **Issue対応（機能追加・デザイン変更などの開発作業）は作業ブランチを切ってPRを出す**。mainへ直接pushしない（2026-07-05リュウキ指定）。マージはリュウキのレビュー後
- 非mainブランチのビルドは無効化済み（2026-07-11、issue #18で解決。ブランチpushでは本番に出ない）。その代わりPRにCIチェックは付かないので、push前のローカル `npm run build` が唯一の検証になる
- 開発作業は、ローカルで `npm run build` と `npm run preview` で確認を済ませ、**意味のあるまとまり単位でpushする**。WIPのこまめなpushをしない
- 複数の小さな変更が続きそうなときは、ローカルで積んでからまとめてpushする
- **npmパッケージを追加・更新したら、pushする前に必ず次を実行する**（npmの既知問題で、Macでの `npm install` はLinux向けオプショナル依存をlockに書き込まないことがあり、CIの `npm ci` が落ちる。2026-07-04に2回発生）

```bash
rm -rf node_modules package-lock.json && npm install   # lockを完全再生成
rm -rf node_modules && npm clean-install               # CIと同じ手順の再現確認
npm run build
```

## 記事の書き方

- 記事は `src/content/<コレクション>/` に .md で置く。コレクションは tech / travel / memo / stream / slides（詳細はREADME）
- 登壇スライドは slides コレクションに置く（frontmatter: `speakerdeckId` 必須、`event` / `eventUrl` / `pdfUrl` 任意。Speaker Deckの共有画面にある `speakerdeck.com/player/<ID>` の32桁英数字がID）。プレイヤーURLを本文に単独行で書くと任意の記事内でもiframe展開される。通常のデッキURLは従来どおりリンクカード（実装: src/lib/remark-speaker-deck.mjs、経緯: issue #38 / PR #39）。**ただしslidesの入口（ヘッダーのナビ・トップのls -l行）はいったん非表示中**（2026-08-15リュウキ判断、当面は記事内埋め込みのみ運用。/slides/ 自体は生成される。再表示は Header.astro と index.astro の2箇所を戻すだけで、そのときの定位置はナビ・ls -l行とも最後尾＝Streamの後ろ、2026-08-15リュウキ指定）
- frontmatterのスキーマは `src/content.config.ts` が正
- `draft: true` の間は一覧・RSS・ページ生成から除外される
- **ローカル下書きは `src/content/<コレクション>/private/` に置く**（gitignore済み、issue #31）。dev・ローカルビルドでは `/tech/private/<basename>/` として一覧・記事ページ・OG画像込みで実サイトの見た目でプレビューできるが、gitに載らないため本番には出ない（publicリポジトリなのでdraft運用より確実）。検索窓にだけは出ない（search-index.mjsがコレクション直下のみ読むため）。公開時はファイルをコレクション直下へ移動するだけ。削除した下書きが「幻のページ」としてローカルビルドに残り続けたら、Astroのcontent layerキャッシュが原因なので `rm -rf node_modules/.astro` で解消（2026-08-15に発生）
- Qiita互換の `:::note` / `:::note warn` / `:::note alert` 記法が使える（実装: src/lib/remark-qiita-note.mjs）
- streamコレクションはfrontmatterに `format: dialogue` を書くと `@speaker: ` 記法の対話ログ形式で表示される。地の文は `@note:`（記法の見本: src/content/stream/dialogue-format-sample.md、実装: src/lib/remark-dialogue.mjs、経緯: issue #12 / PR #13）
- URLだけの行はリンクカードに展開される（note内でも可、URLの前後に空行が必要）。インラインリンクと画像URLは変換されない
- ```mermaid ブロックはブラウザ側でSVGに描画される（記事ページ限定・図があるページだけmermaid.jsを遅延読み込み。実装: src/layouts/BlogPost.astro のscript）
- 画像はリポジトリに置かず、外部CDN（images.ryu-ki-learn.com）のURLを `![説明](URL)` で参照する
- 記事ごとのOG画像（SNSシェア時のサムネ）はビルド時に自動生成される（`/og/<コレクション>/<id>.png`、実装: src/lib/og-image.ts + src/pages/og/）。frontmatterで `heroImage` を設定した記事はそちらが優先。記事以外のページはサイト共通の og-default.png
- tech以外（travel / memo / stream）のカジュアル投稿は `post-blog` スキルで行える。Claude Codeでは `/post-blog`、Codexでは `$post-blog` として明示的に呼び出せる。Discord（道草/ざっくりメモ/チラ裏）・obsidian・会話テキストなど任意の入り口から、誤字脱字の自動修正→一括確認→mainへのpushまでを一括実行する（正本: `.claude/skills/post-blog/SKILL.md`、Codex向けリンク: `.agents/skills/post-blog`、経緯: issue #8）

## よく使うコマンド

- `npm run dev` — 開発サーバー（http://localhost:4321）。バックグラウンド起動は `astro dev --background`（`astro dev stop` / `status` / `logs` で管理）
- `npm run build` — 本番ビルド（push前の確認に必ず通す）
- `npm run preview` — ビルド結果のローカル確認
- `node scripts/generate-og.mjs` — OG画像・favicon PNGの再生成（サイト説明を変えたら実行）

## 規約

- コミットメッセージは1行の日本語でシンプルに
- デザインの方向性は「道草 × .log」（生成りの紙 + 草色 + 等幅フォントのターミナル感）。テーマはCSS変数（src/styles/global.css）で管理し、ダークモードは prefers-color-scheme で自動切替
- サイト名の由来はサイト上に書かない

## 現在の状況

最終更新: 2026-10-18（DuckDB Wasm 検索の記事を tech へ個別移植）

- 直近の動き（2026-10-18）: **DuckDB Wasm 検索の記事「【DuckDB】ブログの記事検索を DuckDB Wasm + Parquet で作ってみた」を tech へ個別移植**（`--only duckdb-wasm-blog-search --pub-date 2026-10-18`、slug-map に追加。画像 2 枚は CDN 既存のため転送なし、views:18 引き継ぎ）。これで限定共有中の記事の未移植はゼロ

- 直近の動き（2026-10-13）: **Strands Decider 記事「【Strands Decider】LM Studio と組み合わせてエージェントの判断までローカルで完結させてみる」を tech へ個別移植**（`--only strands-decider-lmstudio --pub-date 2026-10-07`、slug-map に追加。画像 1 枚は CDN 既存のため転送なし、views:544 引き継ぎ）

- 直近の動き（2026-09-29）: **otel-lgtm 記事「【OpenTelemetry】grafana/otel-lgtm から OTel 入門してみたい」を tech へ個別移植**（`--only grafana-otel-lgtm-anatomy --pub-date 2026-09-28`、slug-map に追加。画像 3 枚は CDN 既存のため転送なし、views:79 引き継ぎ）。migrate.mjs の importedDate / fetchedAt は UTC の日付になるため、JST の日付（2026-09-29）に手で直した。限定共有中の残りは duckdb-wasm-blog-search の 1 本

- 直近の動き（2026-09-27）: **Qiita で本公開済みだった未移植の 3 本を tech へ個別移植**（agentcore-handson-env-design `--pub-date 2026-09-07` / github-username-change-oidc-immutable-sub `--pub-date 2026-09-14` / typesafe-jev-ops `--pub-date 2026-09-18`、slug-map に追加。画像は CDN 既存のため転送なし）。agentcore-handson-env-design は冒頭の Qiita リレー告知を mcks.log 側で削除（エンドレスエイト記事と同じ扱い）。**本公開日は Qiita の記事ページの HTML に埋め込まれた `publishedAt`（UTC）で確かめられる**（API v2 の created_at は限定共有した日のまま）。これで本公開済み Qiita 記事の未移植はゼロ。限定共有中は duckdb-wasm-blog-search と grafana-otel-lgtm-anatomy の 2 本

- 直近の動き（2026-09-27）: **Wasm 概要記事「【Wasm】WebAssemblyとは？〜ひとまずざっくり調べてみた〜」を tech へ個別移植**（`--only wasm-overview --pub-date 2026-09-25`、slug-map に追加。画像は CDN 既存のため転送なし、views:234 引き継ぎ）。限定共有中の残り 1 本（duckdb-wasm-blog-search）も Qiita 本公開後に同じ手順で移植する
- 直近の動き（2026-09-07）: **GitHub ユーザー名を ryuki-imachi から umitsu-tech に変更**。リポジトリは https://github.com/umitsu-tech/mcks.log になった（旧 URL はリダイレクトされるが、旧名が第三者に取られる前提で参照は新名に揃える）。記事内のリポジトリリンクと clone コマンドを新名に更新（issue #50 / PR #51、tech 4 本: devcontainer-python-setup / devops-agent-fis-handson / langfuse-selfhost-ec2-low-cost / ollama-strands-agui-local-agent）。マージ後 90 秒で本番に反映され、**Cloudflare Workers Builds の GitHub 連携はユーザー名変更後も再接続なしで動くことを確認**。手順と経緯は discord-workspace/docs/github-org-rename.md
- 直近の動き（2026-09-07）: **記事内画像の拡大表示（issue #46）と、拡大中のズーム・移動（issue #48）**。拡大した画像はホイール / ピンチで等倍〜8 倍（マウスや 2 本指の中点を中心に寄る）、ドラッグ / 2 本指で移動、ダブルクリック / ダブルタップで等倍と 2.5 倍の切り替え。等倍で画像を押すと閉じ、拡大中は × か Esc か画像の外側で閉じる。Pointer Events で自前実装（`touch-action: none`）。本文の画像（`.prose img:not([class])`）とヒーロー画像を押すと画面いっぱいのオーバーレイに表示し、もう一度押すか × か Esc で閉じる。ライブラリ無しで `src/components/ImageLightbox.astro` 1 つに閉じ、`BlogPost.astro` の Footer の後ろに差し込んでいる（SlidePost は BlogPost 経由で効く）。オーバーレイは position: fixed がレイアウトの枠に閉じ込められないよう、スクリプトで body 直下へ移している
- 直近の動き（2026-09-02）: **エンドレスエイト×AgentCore Memory記事をQiita本公開当日にtechへ個別移植**（`--only endless-eight-agentcore-memory --pub-date 2026-09-02`、slug-mapに追加。画像はCDN既存のため転送なし）。同日、**memo記事「Enterprise Frontier Safeguards（EFS）とは？」を投稿**（`src/content/memo/enterprise-frontier-safeguards.md`、図3枚はCDN `enterprise-frontier-safeguards/` 配下。Claude Fable 5.1発表に合わせた解説で、Discord #ざっくりメモ のスレッド経由・AI執筆＋レビュー全件反映。drawio原本はdiscord-workspace/tmp-diagrams/efs-*）
- 直近の動き（2026-08-29）: **Qiita本公開済みの未移植2本を個別移植**（ハイクの日俳句記事 `--only claude-haiku-575 --pub-date 2026-08-20` / AgentCore Runtime命名記事 `--only agentcore-runtime-naming-rules --pub-date 2026-08-25`、いずれも画像なし・tech行き）。これで**本公開済みQiita記事の未移植はゼロ**。限定共有中の2本（wasm-overview / duckdb-wasm-blog-search）はQiita本公開後に `--only <basename> --pub-date <本公開日>` で個別移植する（issue #35の運用）

- 完了済み: サイト構築フェーズ完了、本運用開始。機能はpush→自動デプロイ、Qiita互換:::note記法、リンクカード、Mermaidレンダリング、4コレクション（tech/travel/memo/stream。旧othersを2026-07-05にstreamへリネーム）、スクロール追従サイド目次（1120px以上のみ表示）、記事内画像の枠線（色は --img-frame で管理）、記事別OG画像のビルド時自動生成（PR #6、詳細は「記事の書き方」参照）、stream対話ログ形式（format: dialogue、@speaker:記法。issue #12 / PR #13、詳細は「記事の書き方」参照。デザイン経緯と改善余地はissue #12に記録）、**Qiita過去記事の一括移植完了（issue #16 / PR #17、2026-07-11）**: 116本（tech 108 / stream 8）、画像337枚を自CDNへ移行、スクリプトは scripts/qiita-migration/（未公開の残り2本はQiita側で本公開後に `--only <basename> --pub-date <本公開日>` で個別移植する。限定共有を経た記事はcreated_atが限定公開日のため本公開日の指定が必要、issue #35）、**一覧のページネーション（issue #19 / PR #20、2026-07-11）**: 1ページ20件、Qiita/Zenn風の数字並び+現在ページ円形塗り、10ページ以上で省略記号（実装: src/components/Pagination.astro + 各 [...page].astro）、**Cloudflare Web Analytics導入（issue #15 / PR #21、2026-07-11）**: BaseHead.astroにビーコンタグ、閲覧状況はCloudflareダッシュボードで見る、**記事検索（issue #14 / PR #22、2026-07-11）**: 全コレクション横断の全文検索。ビルド時にsearch-index.parquetを生成（src/integrations/search-index.mjs）し、DuckDB Wasm（jsDelivr CDNから遅延ロード。wasmが30MB超でWorkersアセットの25MB上限を超えるためセルフホスト不可、PR #25で対応）がブラウザでSQL検索。一覧4ページ+トップの検索窓、日付降順、ヒット語ハイライト+本文スニペット付き（実装: src/lib/search-client.ts / src/components/SearchBox.astro）、**Xシェアボタン（issue #2 / PR #23・#27・#28、2026-07-11〜12）**: 記事ヘッダーのタグ行右端のピル型ボタンからWeb Intentを開く（タイトル+URL+#mckslog、API・JS不要。実装: src/components/ShareOnX.astro。配置は本文末尾→左レールを試してタグ行右端に確定）、**カジュアル投稿スキル /post-blog（issue #8 / #3 / PR #24、2026-07-11）**: tech以外の投稿パイプライン（詳細は「記事の書き方」参照）、**記事ページ本文の可変幅レイアウト（issue #10 / PR #29、2026-07-12）**: Qiita/Zenn方式の可変幅にして目次が残る画面幅を拡大、**ローカル下書きのprivateディレクトリ規約（issue #31 / PR #32、2026-07-12）**: 詳細は「記事の書き方」参照、**トップのセクション入口のls -l出力風化（issue #1 / PR #34、2026-07-22）**: 枠線・角丸を全廃して記事一覧と同じ下線区切り行に統一、posts数は右揃えtabular-nums・0件セクションは薄字empty表記（検討経緯・見送り案はissue #1のコメント参照。issueは2026-07-22にクローズ済み）、**スライド投稿機能（issue #38 / PR #39、2026-08-15）**: slidesコレクション（一覧・詳細・RSS・OG画像・検索対応）+ Speaker Deck埋め込み（詳細は「記事の書き方」参照。実装Codex・レビューClaude Codeの初の分業PR）
- 継続検討: issues参照（#4 検索エンジン未インデックス、#26 検索エンジンの初回ロード時間改善。#26はドラフトPR #30（feature/search-engine-warmup）を2026-07-13から保留中——現状の本番挙動で困っていないためマージせずopenのまま。実測値・設計判断の記録はPR #30本文参照）
- 次の一歩: stream対話ログ形式での1本目の記事を post-blog スキルで投稿（題材候補: CB特典クレジットの使い道、Discordでの相談返事待ち）
- 運用メモ: Issue対応はブランチ+PR（「pushの運用ルール」参照）
- Cloudflare設定の2層管理: **Worker設定**（observability等）はwrangler.jsoncで宣言管理（ダッシュボードで変えない。デプロイで上書きされるため）。**ゾーン設定**（ドメイン側: Caching / Security / Email等）はwranglerの管轄外なのでダッシュボード操作とし、変更したらここに記録する。有効化済み: Workers Observability（2026-08-16、PR #41）、Crawler Hints（2026-08-16、issue #4対応の一部。IndexNow対応エンジンへ更新を自動通知）
- 関連リソース: 記事の下書き・構築ログ・設計書はリポジトリ外の作業ディレクトリで管理（Discordの #garage / #記事の種 スレッドで進行）

## Astroドキュメント

作業前に関連ガイドを参照する: https://docs.astro.build

- [ルーティング](https://docs.astro.build/en/guides/routing/)
- [Astroコンポーネント](https://docs.astro.build/en/basics/astro-components/)
- [コンテンツコレクション](https://docs.astro.build/en/guides/content-collections/)
- [スタイリング](https://docs.astro.build/en/guides/styling/)
