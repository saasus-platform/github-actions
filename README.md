# saasus-platform/github-actions

SDK 自動テスト（契約テスト）の共通CI基盤。保守は1か所（本リポジトリ）、実行は各SDKリポジトリ、という方針の土台です。
背景・目的・設計判断は社内の設計ドキュメントを参照してください。

本リポジトリは **private** ですが、Actions の Access を「org 内から参照可」に設定することで、公開SDKリポジトリの
ワークフローから `uses:` で参照できます（HANDOFF 参照）。

## 担保する3観点

- **(A)** 最新 OpenAPI から最新SDKを生成し、ビルド/型チェックが通る
- **(B)** 最新SDKのリクエスト/レスポンスが仕様準拠（Prismが受理し、応答がデシリアライズできる）
- **(C)** 旧SDK（最新の安定版gitタグ）が、フィールド追加された新レスポンスで壊れない（前方互換）

## 構成（なぜコンポジットアクションか）

言語非依存の共通ステップのみをコンポジットアクションとして提供します。ビルド/型チェックや
テストハーネス本体は言語依存のため各SDKリポジトリ側に置きます。

`workflow_call`（再利用可能ワークフロー）を採用しないのは、Prism モックとテストハーネスを
**同一ジョブ・同一ランナー**で動かす必要があるためです。`workflow_call` は呼び出し先が別ジョブ・別VMになり、
バックグラウンド起動した Prism に `localhost` で到達できません。コンポジットアクションは呼び出し側ジョブの
ステップ列にインラインで展開されるため、同一ランナーを共有できます。

## コンポジットアクション 入力/出力インターフェース

### `resolve-latest-stable-tag`
対象リポジトリの最新の安定版（非rc）タグ `^v[0-9]+\.[0-9]+\.[0-9]+$` を解決します。

| 入力 | 必須 | 既定 | 説明 |
|---|---|---|---|
| `repository` |  | 呼び出し元リポジトリ | `owner/repo` 形式。省略時は `$GITHUB_REPOSITORY`（自リポジトリ） |
| `token` |  | `""` | private リポジトリ参照時のみ必要（contents:read）。ログには出力しない |

| 出力 | 説明 |
|---|---|
| `tag` | 解決したタグ（例 `v1.13.0`） |

### `fetch-openapi-spec`
api リポジトリ（**固定**: `Anti-Pattern-Inc/saasus-api`）から対象モジュールの OpenAPI 定義のみを取得します（全クローンしない）。
`ref` 省略時は api の最新の安定版（非rc）タグを**このアクション内で自動解決**します。
取得物はワークスペース内限定。トークン・spec本文はログに出力しません。

| 入力 | 必須 | 既定 | 説明 |
|---|---|---|---|
| `ref` |  | 最新の非rcタグ | 取得する git ref。省略時は api の最新安定版タグを自動解決 |
| `token` | ✓ | — | api への contents:read（GitHub App installation token） |
| `modules` |  | 7モジュール | 空白区切りのモジュール一覧 |
| `dest` |  | `openapi-spec` | 出力先（ワークスペース相対） |

| 出力 | 説明 |
|---|---|
| `spec-dir` | `<module>.yml` を格納したディレクトリの絶対パス |
| `ref` | 実際に使用した api ref（省略時は解決されたタグ） |
| `modules` | 取得したモジュール一覧（`setup-prism` にそのまま渡せる） |

ファイル名は `<module>.yml` に正規化します（api 側の `integration.yml` も `integration.yml` のまま、
他は `<module>api.yml` → `<module>.yml`）。各言語の生成ツールが別名を要求する場合は、呼び出し側で
マッピングしてください（Java の例はテンプレート参照）。

### `setup-prism`
モジュールごとに Prism モックを静的モード・`--errors` で起動します。各サーバは `127.0.0.1` の連番ポートで待受。
Prism 自身のログはファイルへリダイレクトし、**成功・失敗いずれの場合も**応答本文を workflow ログに出しません。起動前に対象ポートが空いていることを確認し、readiness では **そのポートを LISTEN しているのが起動した Prism プロセス自身（またはその子）であること**を `ss` で検証します。これにより、別サーバの応答や、自Prismが spec 読込中にポート競合で後から落ちるケースを誤って成功と判定しません。

| 入力 | 必須 | 既定 | 説明 |
|---|---|---|---|
| `spec-dir` | ✓ | — | `fetch-openapi-spec` の出力ディレクトリ |
| `modules` |  | 7モジュール | 起動順＝ポート割当順。`fetch-openapi-spec` の `modules` 出力を渡すと一覧の重複定義を避けられる |
| `base-port` |  | `4010` | N番目のモジュールは `base-port + N`（0始まり） |
| `prism-version` |  | `5.8.1` | 固定する `@stoplight/prism-cli` のバージョン |

| 出力 | 説明 |
|---|---|
| `endpoints-file` | `MODULE=URL` 形式の行を持つファイルのパス |

さらに次を job 環境へエクスポートします:
- `PRISM_URL_<MODULE>`（例 `PRISM_URL_AUTH=http://127.0.0.1:4010`）
- `PRISM_ENDPOINTS_FILE`（上記ファイルのパス）

> SDK の baseURL は spec の `servers` のパス（例 `/v1/auth`）を含めず、各 Prism のルート
> （`http://127.0.0.1:<port>`）に設定してください。Prism は `paths:` をルート直下で配信します。

## 呼び出し規約（各SDKリポジトリ側）

- トリガーは `on: pull_request`（対象 `main`、`opened/synchronize/reopened`）。`pull_request_target` は使わない。
- 契約テストのジョブ名を固定し、`main` のブランチ保護で **required status check** にする。
- fork からの PR には secrets が渡らないため、完全テストは同一リポジトリ内ブランチの PR で実行する。

呼び出しワークフローとテストハーネスは各SDKリポジトリ側に配置します（言語別のテンプレートは各SDKリポジトリを参照）。

## 安全規律

- トークン/Bearer は `::add-mask::` でマスク。`set -x` やsecretsのechoを禁止。
- 応答本文・spec本文・PII をログ出力しない。失敗時も HTTP ステータスコードのみ。
- 合成データのみ使用（Bearer は合成トークン、Prism 応答はスキーマ由来の合成値）。
- private spec はワークスペース内限定。アーティファクトとして公開しない。

## Pinning（必須チェック化の前に）

テンプレートはサードパーティ/公式アクションを可読性のため `@v4`/`@v1` のバージョンタグで参照しています。
required check として有効化する前に、**コミットSHAで固定**してください（Dependabot/actionlint 推奨）。
`uses: saasus-platform/github-actions/...@main` も、安定後はタグ/SHA 固定を検討してください。
