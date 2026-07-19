# Approved Browser MCP

Playwright MCPの上に、承認・権限・監査の境界を追加するローカルMCPポリシーレイヤーです。

> Human signs in. AI acts with approval.

## 目的

- Playwright MCPをブラウザ操作の実行エンジンとして利用する
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

ブラウザ操作本体はPlaywright MCPに委譲します。Chrome DevTools MCPなどへの対応は、必要性が確認された場合の将来拡張です。

## AIris MCP Gatewayとの関係

このサーバーは単独で利用できる無料OSSです。AIris MCP Gatewayでは、Playwright MCPを非公開の子プロセスとして所有し、クライアントにはこのサーバーだけを公開します。Gateway本体へ承認ロジックやブラウザ制御コードはコピーしません。

組織全体のSSO、RBAC、ポリシー配布、長期監査、SIEM連携などの企業向け統制はAIris OSの責務です。MCP本体を有料サービス化することは目的にしません。

詳細は [設計書](docs/design.md) を参照してください。
