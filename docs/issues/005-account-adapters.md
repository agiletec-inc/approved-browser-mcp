# adapters: 名前付きアカウント照合と高水準操作を追加する

## Problem / outcome

「X公式アカウントから投稿」のような操作では、ブラウザプロファイル名だけでは実際のログインアカウントを保証できない。DOM操作ではなく、対象サイト・アカウント・結果を検証する高水準操作が必要である。

## Evidence and current behavior

- `docs/design.md` §4・§5 は名前付きブラウザIDとcommit直前の実アカウント照合を定義している。
- `docs/adr/ADR-0001-approved-policy-layer.md` はアカウント照合を決定している。
- 現在はサイトアダプターもアカウント識別も未実装である。

## Reproduction or baseline

テストサイトに2つのログインアカウントを用意し、同じプロファイル名または誤ったアカウント状態でcommitを試行する。アカウント識別不能・不一致を拒否する。

## Expected behavior

アダプターは現在のoriginとログインアカウントを検証し、高水準のprepare/commit操作を提供する。アカウント不一致、識別不能、結果URL・本文の検証失敗ではcommitを完了扱いにしない。

## Scope

- アダプター契約（account identity、prepare、commit、result verification）
- 名前付きアカウント設定
- まず1サイトの実装（候補: XまたはReddit）
- receiptへのアカウント・結果検証

## Non-goals

- 複数サイトの同時実装
- 認証回避・内部API利用
- 組織横断のアダプター配布（利用側プロダクト）

## Acceptance criteria

- [ ] AC-1: Given a named account configuration, when the adapter checks the active session, then the observed account identity must match before commit is allowed.
- [ ] AC-2: Given an account mismatch or unknown identity, when commit is requested, then it is rejected without sending a backend commit.
- [ ] AC-3: Given a successful high-level action, when the site result is verified, then the receipt contains the named account, observed identity, result URL, and verification timestamp.
- [ ] AC-4: Given a site adapter that does not support an operation, when the operation is requested, then no generic DOM fallback is attempted.

## Verification plan

| Criterion | Verification | Expected |
|---|---|---|
| AC-1 | `pnpm test --filter account-adapter` | matching identity allows prepare/commit |
| AC-2 | `pnpm test --filter account-adapter` | mismatch/unknown rejects without backend call |
| AC-3 | `pnpm test --filter account-adapter` | verified receipt fields present |
| AC-4 | `pnpm test --filter account-adapter` | unsupported operation fails closed |

## Risks, dependencies, rollout

- Dependencies: Issues #2, #3, and #4.
- Platform policies differ; each adapter requires a documented allowed-operation boundary.
- Start with one adapter and keep others disabled.

## Clarifications and assumptions

- The first site is selected during implementation based on available test account and stable identity surface.
