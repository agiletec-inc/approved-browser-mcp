# ADR-0001: Approved Browser MCPをブラウザ操作のポリシー層として実装する

- 状態: Accepted
- 日付: 2026-07-19

## コンテキスト

Playwright MCPは、ブラウザ操作、永続プロファイル、ログイン済みセッション、既存ブラウザ接続をすでに提供している。ブラウザ操作自体を再実装すると、保守負債と既存OSSとの重複が発生する。

一方、ログイン済みブラウザを実アカウントで使うには、操作の意味に基づく承認、アカウント分離、公開データ制御、監査、プロンプトインジェクション境界が必要である。

## 決定

`approved-browser-mcp`はPlaywright MCPの前段に置く無料OSSのポリシー層とする。

- ブラウザ操作本体はPlaywright MCPへ委譲する
- Playwright MCPは`approved-browser-mcp`の非公開子プロセスとして起動する
- クライアントには`approved-browser-mcp`だけを公開する
- 操作を`observe` / `interact` / `commit`に分類する
- `commit`は`prepare → user approval → commit → receipt`とする
- 承認はAI向けMCPツールから呼べないローカルまたは信頼済みユーザー経路で行う
- 名前付きアカウントと実ログインアカウントをcommit直前に照合する
- 組織SSO、RBAC、ポリシー配布、長期監査、SIEM連携はAIris OSの責務とする
- AIris MCP Gatewayは発見、ルーティング、ライフサイクルだけを担当する

## 却下した選択肢

- ブラウザドライバーを独自実装する: Playwright MCPと重複するため
- Gatewayに承認ロジックを入れる: Gatewayの責務を超えるため
- AIへ`browser_action_approve`を公開する: AI自身による承認迂回を許すため
- 汎用クリック・入力を既定で安全扱いする: UIの意味を一般には保証できないため

## 結果

個人利用ではローカル無料OSSとして配布でき、企業向けの統制機能はAIris OSへ自然に拡張できる。代わりに、Playwright MCPのプロセス所有境界、ローカル承認経路、サイト固有のアカウント照合が実装上の重要な依存になる。
