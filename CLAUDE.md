# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

アーキテクチャ・レイヤー規約・新規ページ追加のチェックリスト・セキュリティ上の不変条件は AGENTS.md に網羅されている（`GEMINI.md` はその symlink）。ここには AGENTS.md / README.md に無い補足だけを書く。

@AGENTS.md

## リポジトリの位置づけ

- `GoogleChromeLabs/project-sesame` のフォーク（`origin` は `seotaro/seotaro-sesame`、`upstream` リモートは未設定）。ソース・ドキュメントは upstream に合わせて英語で書かれているので、コード内のコメントや help Markdown も英語で揃える。
- Android との連携点は `associated_domains`（各 `*.config.json`）→ `GET /.well-known/assetlinks.json`（`src/server/middlewares/well-known.ts`）による Digital Asset Links と、`src/server/config.ts` の `generateApkKeyHash()`（SHA-256 証明書フィンガープリント → `android:apk-key-hash:` origin）。

## 自前環境 `seotaro`（Android の Shrine 用）

`../android-identity-samples/Shrine` が接続する自前サーバー。GCP プロジェクト `seotaro-sesame`（App Engine, `asia-northeast1`）。

- 設定は `seotaro.config.json` + `seotaro.yaml`（`NODE_ENV=seotaro`）。セッションの `secret` は gitignore 済みの `seotaro.secret.config.json` に置き、`src/server/config.ts` が `<NODE_ENV>.secret.config.json` があれば最後にマージする（fork で追加した仕組み）。
- App Engine 側ではビルドされず手元の `dist/` がそのまま上がるので、デプロイは `npm run deploy:seotaro`（build → `gcloud app deploy seotaro.yaml cron.yaml`）。
- `hostname`・Shrine の `API_BASE_DOMAIN`・Shrine の `asset_statements`（`strings.xml`）・`associated_domains` の SHA-256 は互いに一致していないと passkey / Restore Credentials が失敗する。どれかを変えるときは両リポジトリを同時に直す。
- cron が作成から 48 時間経ったアカウントを消すので、長期間使うユーザー名は `allowlisted_accounts` に入れる。

## コマンド補足

- 前提: Node.js 24.7+、Firestore emulator 用に Java 21+。CI は `npm ci --legacy-peer-deps` でインストールしている。
- 型チェック: `npm run typecheck`（ルート・`src/client`・`src/server` の 3 つの tsconfig を順に検査）。`npm run check` に含まれる。
- テストは `firebase emulators:exec` 経由でないと Firestore に繋がらない。単一ファイル／単一テストを走らせるには:

  ```shell
  NODE_ENV=localhost npx firebase emulators:exec --only firestore "npx vitest run src/server/libs/evp.test.ts"
  NODE_ENV=localhost npx firebase emulators:exec --only firestore "npx vitest run src/server/libs/evp.test.ts -t '<テスト名>'"
  ```

- テストは `src/**/*.test.ts` に対象ファイルと並べて置く。`vitest.config.ts` で coverage が常時有効になっており、`~project-sesame/{client,server,shared}` エイリアスもここで解決される。
- `npm run deploy` / `deploy:idp` / `deploy:prod`（GAE への `gcloud app deploy`）と `npm run indexes`（Firestore インデックスの deploy）は実環境に反映される。ユーザーの指示なく実行しない。`npm run dev:local` は `caddy.sh` で `sudo` を要求する。

## 設定ファイルのマージ挙動（`src/server/config.ts`）

- マージ順は `default.config.json` → （`NODE_ENV` が `rp-localhost` / `idp-localhost` のときだけ）`localhost.config.json` → `<NODE_ENV>.config.json`。`NODE_ENV` 未設定は `localhost`、`test` も `localhost` として扱われる。
- 配列は上書きではなく **和集合** になる。`supported_idps` / `supported_rps` は `origin`、`associated_domains` は `site` または `package_name` をキーに後勝ちで重複排除され、それ以外の配列は値で dedupe される。したがって環境別 config で default の要素を消すことはできない。
- IdP として動くかどうかは `NODE_ENV` が `idp` / `idp-localhost`、または `project_name === 'sesame-identity-provider'` で決まる。
- Firestore emulator のシードデータは `./.data`（gitignore 済み）。`npm run emulator` 終了時に書き戻される。

## help Markdown を書くとき

`src/shared/helps/<slug>.{usage,dev}.md` を編集する際は、`.agents/skills/usage-helper-writer/SKILL.md` と `.agents/skills/dev-helper-writer/SKILL.md` を読んでその指示に従う（Claude Code のスキルとしては自動登録されない）。
