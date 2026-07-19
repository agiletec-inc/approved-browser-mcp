# Approved Browser MCP

Playwright MCPの上に、承認・権限・監査の境界を追加するローカルMCPポリシーレイヤーです。

> Human signs in. AI acts with approval.

## 目的

- Playwright MCPをブラウザ操作の実行エンジンとして利用する
- 専用ChromiumはローカルDocker内で起動する
- ユーザーがログインした専用ブラウザプロファイルを、個人ブラウザから分離する
- `observe` / `interact` / `commit` を操作の影響度で分類する
- 投稿、購入、予約、削除などを `prepare → user approval → commit` にする
- クライアント、プロファイル、ドメイン、公開データ種別を制限する
- 実行前後を監査できるようにする
- Semantic操作を基本にし、必要な画面だけVisionへフォールバックする
- 個人利用は完全ローカルの無料OSSとして提供する

## 独自に持たないもの

- ブラウザドライバー
- Chromium起動・Cookie実装
- サイトごとのスクレイパー
- 認証・課金・CAPTCHAの回避

ブラウザ操作本体はPlaywright MCPに委譲します。専用ChromiumとPlaywright MCPはローカルDocker内で動かし、Chrome DevTools MCPなどへの対応は将来拡張とします。

## AIRIS MCP Gatewayとの関係

このサーバーは単独で利用できる無料OSSです。AIRIS MCP Gatewayは、汎用MCPの登録・発見・ルーティング・ライフサイクルを担う独立OSSです。必要に応じて、AIRIS MCP Gatewayからapproved-browser-mcpを公開・起動できます。

AIRIS MCP Gateway本体へ承認ロジックやブラウザ制御コードはコピーしません。Playwright MCPの非公開化とブラウザ固有ポリシーはapproved-browser-mcpの責務です。

組織全体のSSO、RBAC、ポリシー配布、長期監査、SIEM連携などは、AIris OSやAIris Suiteなどの利用側プロダクトが必要に応じて所有します。AIRIS MCP Gatewayやapproved-browser-mcpを有料の企業統制基盤として扱うことは目的にしません。

詳細は [設計書](docs/design.md) を参照してください。
