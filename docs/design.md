# Approved Browser MCP 設計書

## 1. 概要

Approved Browser MCPは、Playwright MCPの前段に配置するローカルポリシーレイヤーである。ブラウザ自体を作るのではなく、AIクライアントからの操作を検査し、必要な場合だけユーザー承認を要求してPlaywright MCPへ渡す。

キラーフィーチャーは「ログイン済みブラウザ」ではなく、**AIがブラウザを操作しても、外部状態を変える操作はユーザー承認なしに実行できないこと**である。

## 2. 解決する課題

Playwright MCPは、永続プロファイルや既存ブラウザへの接続を提供する。一方、ブラウザ操作MCP単体は、ログイン済みページの内容をAIへ公開する範囲、危険操作の承認、サイト・アカウント単位の権限、監査を一貫した業務境界として提供しない。

Approved Browser MCPはこの不足部分だけを埋める。

## 3. システム構成

```text
Claude Code / Claude Desktop / Cursor
                 │ MCP
                 ▼
       approved-browser-mcp
        ├─ Client / Profile Policy
        ├─ Domain Allowlist
        ├─ Action Classifier
        ├─ Approval Manager
        ├─ Output Filter
        ├─ Prompt Injection Boundary
        ├─ Profile Lock
        └─ Audit Log
                 │ backend MCP
                 ▼
       Playwright MCP
                 │
                 ▼
       専用Chromiumプロファイル
```

AIris MCP Gatewayへ統合する場合:

```text
AI client
   ▼
AIris MCP Gateway
   ├─ discovery / lifecycle
   ├─ approved-browser-mcp
   └─ Playwright MCP
          ▼
      host-native Chromium
```

Gatewayはプロセスの発見・起動・停止だけを担当する。承認判断とブラウザ操作のポリシーはこのリポジトリに置く。

## 4. 責務境界

### Approved Browser MCPが所有するもの

- MCPクライアントごとのアカウントID・プロファイル・ドメイン・公開レベル設定
- 名前付きブラウザID（例: `x-agiletec-official`、`instagram-recruiting`）
- 操作の `observe` / `interact` / `commit` 分類
- `prepare → user approval → commit` の承認フロー
- 一回限り・短時間有効の承認トークン
- Cookie、Authorization、パスワード等の出力除外
- 本文・スクリーンショットの公開レベル
- プロンプトインジェクション境界
- プロファイル単位の排他ロック
- 監査ログ

### Playwright MCPが所有するもの

- Chromiumの起動・接続・終了
- Playwrightによるページ操作
- Cookie・セッションのブラウザ内保持
- DOM、アクセシビリティツリー、スクリーンショットの取得

### AIris MCP Gatewayが所有するもの

- MCPサーバーの登録・発見・ルーティング
- COLD起動とアイドル停止
- クライアント接続設定
- プロセスのヘルスチェック
- Playwright MCPを非公開の子プロセスとして起動するライフサイクル

### AIris OSが所有するもの（将来の企業統制）

- 組織全体のポリシー配布
- SSO、RBAC、端末・ユーザー管理
- 承認者・承認経路の組織設定
- 長期監査ログ、検索、SIEM連携
- サイト別の業務アダプター配布とサポート

## 5. 操作モデル

HTTPメソッドではなく、外部状態への影響で分類する。

| レベル | 例 | 承認 |
|---|---|---|
| `observe` | 本文、リンク、画面、現在URLの取得 | ポリシー許可 |
| `interact` | 検索語入力、タブ切替、展開、ページ内移動 | サイト・操作別設定 |
| `commit` | 投稿、返信、DM、購入、予約、削除、設定変更 | 毎回必須 |

`GET`やクリックでもログアウト、削除、予約などが起き得るため、HTTPメソッドだけで安全とは判断しない。

`commit`のフロー:

1. AIが操作内容を準備する。
2. 対象サイト、アカウント、入力値、影響をユーザーへ提示する。
3. ユーザーが承認する。
4. ローカルUI、OSダイアログ、またはAIris OSの信頼済み承認経路でユーザーが承認する。AI向けMCPツールから承認できてはならない。
5. 承認内容に結び付いた一回限りの承認トークンでバックエンドへ渡す。
6. 実行結果と `receipt`（対象、アカウント、結果URL、実行日時、画面証跡）を返し、監査ログへ記録する。

サイト固有アダプターでは、DOM操作を直接公開せず、汎用ツールより明確な意味を持つ `x_post_prepare` / `x_post_commit`、`reddit_comment_prepare` / `reddit_comment_commit` のような操作を提供できる。

## 6. 描画方式と操作フォールバック

React、Next.js、SPA、クライアントサイド遷移、React Server Componentsは、最終的にブラウザへ描画された状態をPlaywrightが操作するため、Reactであること自体は特別な制約にならない。

操作経路は次の優先順位にする。

1. **Semantic mode**: アクセシビリティツリー、ARIA、ラベル、DOM locatorを使う。通常経路であり、最も安定する。
2. **Visual fallback**: スクリーンショットを使う座標クリック、ドラッグ、スクロール。Canvas、WebGL、地図、チャート、独自描画UI向け。
3. **Site adapter**: 投稿、購入、削除などをサイト固有の意味ある操作に変換し、対象アカウントと結果を検証する。

Visionで「押せる」ことは、操作の意味や安全性を保証しない。Visual fallbackのクリック・ドラッグ・入力は原則として `commit` 相当の承認対象にする。サイトアダプターが対象要素、操作、結果検証を明示した場合だけ承認レベルを下げられる。

仮想スクロールやSPAの再描画では、古い要素参照を再利用しない。

```text
scroll → snapshot → 対象探索 → 操作 → 再snapshot
```

投稿などのcommitは、クリック成功だけではreceiptを発行しない。投稿URL、表示された本文、対象アカウントなどの結果を確認してから完了とする。

## 7. データ公開ポリシー

ログイン済みページの本文・DOM・スクリーンショットはAIプロバイダーへ送信され得る。Cookieを外部へ送らないこととは別のリスクとして扱う。

### 公開レベル

| レベル | 返却内容 | 初期値 |
|---|---|---|
| `metadata` | URL、タイトル、日時、件数 | 許可 |
| `text` | 本文と引用位置 | 許可 |
| `screenshot` | 画面画像 | 明示許可 |
| `sensitive` | 個人情報・業務機密の可能性がある内容 | 明示許可 |

クライアントごとに、利用可能なプロファイル、ドメイン、公開レベル、操作レベルを設定する。

承認トークンは、操作内容、アカウントID、実プロファイル、origin、セッションID、有効期限、操作ハッシュに結び付ける。commit直前に、サイトアダプターが現在のoriginとログイン中アカウントを照合し、不一致なら停止する。

## 8. セキュリティ境界（P0）

- プロファイルごとに許可ドメインを明示し、未登録ドメインへ移動しない
- `file://`、`chrome://`、`devtools://`、localhost、Loopback、LAN内IP、クラウドメタデータを既定拒否
- ダウンロード、アップロード、印刷、外部アプリ起動、クリップボードを既定拒否
- 許可ドメイン外へのリダイレクトを拒否
- Cookie、Authorizationヘッダー、パスワード、決済情報、2FAコードをレスポンス・ログに出さない
- プロファイル単位の排他ロックを取り、同時起動で状態を壊さない
- commit直前に名前付きアカウントと実際のログインアカウントを照合する
- ページ内の命令・誘導文を外部コンテンツとしてラベル付けし、AIへの指示として扱わない
- サイトが拒否した場合に、内部API探索、アンチボット回避、別経路への無言フォールバックをしない
- 本文・スクリーンショットがAIプロバイダーへ送信され得ることを接続時に表示する

## 9. MCPツール境界

バックエンドの汎用ツールをそのまま公開せず、ポリシーレイヤーの名前空間で公開する。AIには「どのDOMをクリックするか」ではなく、「どのアカウントから何を実行するか」を扱わせる。

```text
browser_observe_page
browser_observe_links
browser_observe_screenshot
browser_interact_click
browser_interact_type
browser_interact_select
browser_action_prepare
browser_action_commit
browser_action_cancel
browser_user_handoff
browser_session_stop
```

名前付きアカウント操作の例:

```text
x_post_prepare(account="x-agiletec-official", text="...")
x_post_commit(preparation_id="...", approval_token="...")
```

MCPには `browser_action_approve` を公開しない。AIから承認できないようにし、代わりに次の状態確認だけを公開する。

```text
browser_action_status
```

`browser_action_commit` は、ローカルUIまたはAIris OSの信頼済み承認経路で発行された承認トークンなしでは呼び出せない。承認トークンは期限切れまたは一回実行後に無効化する。

GatewayにはPlaywright MCPを別の公開サーバーとして登録しない。`approved-browser-mcp`が非公開stdio子プロセスとして起動し、クライアントから見えるMCPサーバーは`approved-browser-mcp`だけにする。

## 10. セッションとプロファイル

ブラウザプロファイルはバックエンドMCPまたはその拡張機能が管理する。Approved Browser MCPは、プロファイル識別子、利用許可、排他ロック、ライフサイクルを管理するが、Cookieを直接読み出さない。

```text
~/Library/Application Support/approved-browser-mcp/
  policy/
  locks/
  audit/
```

パスワードはMCP引数にせず、ログイン・CAPTCHA・2FA・再認証はユーザーへ引き渡す。

## 11. 実行制限

- 1依頼あたりの最大ページ数: 10
- 1ページあたりの抽出本文: 100 KiB
- 1レスポンスあたりの本文: 500 KiB
- セッションアイドルタイムアウト: 10分
- 1セッションの最大実行時間: 15分
- 自動ページネーション・外部リンク追跡・無限スクロール: 既定無効

## 12. 実装段階

### P0: ポリシー基盤

- MCPバックエンド接続アダプター
- Playwright MCPの非公開子プロセス化
- クライアント・プロファイル・ドメイン許可制
- 危険URL拒否
- プロファイル排他ロック
- 出力フィルター
- プロンプトインジェクション境界
- 監査イベント形式
- AIから呼べないローカル承認経路
- アカウントIDと実ログインアカウントの照合

### P1: observe / interact

- 本文・リンク・スクリーンショット取得
- 汎用クリック・入力・選択は既定で承認対象にする
- アダプターが安全性を明示した操作だけ承認を省略する
- ユーザー引き渡しと再開
- 制限値・タイムアウト

### P2: 承認付きcommit

- `prepare` / 外部承認 / `commit` / `cancel`
- 一回限り承認トークン
- 実行前後の監査ログ
- サイト固有のcommitアダプター

### P3: AIris MCP Gateway統合

- COLDプロバイダー登録（approved-browser-mcpのみ公開）
- host-required capability
- ポリシーMCPとバックエンドMCPの起動・停止
- Claude Code / Claude Desktop接続手順

## 13. 受入条件

- 既存ブラウザMCPをバックエンドとして利用できる
- 個人ブラウザのプロファイルを再利用しない
- 許可ドメイン外・危険URLへ移動できない
- Cookie・パスワード・認証ヘッダーが出力されない
- 本文・スクリーンショットの公開範囲をクライアントごとに制限できる
- Semantic modeを第一経路として利用できる
- Canvas・独自描画UIではVisual fallbackへ切り替えられる
- Visual fallbackの操作が承認なしでcommitにならない
- SPA再描画後に古い要素参照を再利用しない
- ページ内の命令がAIへのシステム指示にならない
- 同じプロファイルを複数セッションで同時利用できない
- commitは承認なしで実行できない
- AI向けMCPツールから承認操作を呼び出せない
- Playwright MCPの生ツールをクライアントから直接呼び出せない
- 汎用interactが既定で外部状態変更を起こさない
- 名前付きアカウントと実ログインアカウントが不一致ならcommitできない
- 承認トークンを再利用できない
- Gatewayなしで単独起動できる
- AIris MCP Gateway経由でCOLD起動できる
