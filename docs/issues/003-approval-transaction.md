# approval: 外部承認付きcommitトランザクションを実装する

## Problem / outcome

AIが投稿、購入、予約、削除などの外部状態変更を自分自身で承認できてはならない。承認内容と実行結果を結び付けた一回限りのトランザクションが必要である。

## Evidence and current behavior

- `docs/design.md` §5・§9 は `prepare → user approval → commit → receipt` を定義している。
- `browser_action_approve` はAI向けMCPツールとして公開しない設計である。
- 現在は承認経路、トークン、receipt、監査ログが未実装である。

## Reproduction or baseline

実装前のため、commit操作は存在しない。テストでは、AI向けMCP呼び出し、期限切れトークン、操作内容を変更したトークン、別アカウントのトークンを拒否する。

## Expected behavior

AIは操作をprepareできるが、承認はローカルUI・OSダイアログ・AIris OSの信頼済み経路でのみ発行される。commitは操作ハッシュ、アカウント、origin、プロファイル、セッション、期限に一致する一回限りのトークンが必要である。

## Scope

- prepare/cancel/status
- AIから呼べない外部承認インターフェース
- 一回限り・短時間有効の承認トークン
- 操作内容と対象アカウントのハッシュ化
- 実行結果receiptと監査イベント

## Non-goals

- 組織の承認者管理やSSO（AIris OS）
- ブラウザ操作・サイト別投稿処理
- 自動承認や承認ダイアログの省略

## Acceptance criteria

- [ ] AC-1: Given a prepared commit, when the AI calls the MCP surface, then it can inspect status but cannot approve the action.
- [ ] AC-2: Given a valid user approval, when commit is requested with matching operation hash, account, origin, profile, session, and unexpired token, then the backend executes exactly once.
- [ ] AC-3: Given a reused, expired, mismatched, or AI-generated approval token, when commit is requested, then the action is rejected and no backend commit is sent.
- [ ] AC-4: Given a successful or failed commit, when the transaction ends, then a receipt and redacted audit event contain the target, account, origin, result, and timestamp.

## Verification plan

| Criterion | Verification | Expected |
|---|---|---|
| AC-1 | `pnpm test --filter approval-flow` | approve tool absent; status available |
| AC-2 | `pnpm test --filter approval-flow` | exactly one backend commit and valid receipt |
| AC-3 | `pnpm test --filter approval-flow` | all invalid-token cases reject without backend call |
| AC-4 | `pnpm test --filter audit-receipt` | receipt/audit fields present and secrets absent |

## Risks, dependencies, rollout

- Dependencies: Issues #001 and #002.
- Security: approval tokens must not be accepted from ordinary MCP tool arguments without a trusted issuer.
- Rollout: keep commit disabled until external approval is wired and tests pass.

## Clarifications and assumptions

- MVP trusted approval channel is local-only; AIris OS integration is a later issuer.
