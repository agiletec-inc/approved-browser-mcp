# browser: Semantic操作とVisionフォールバックをポリシー経由で公開する

## Problem / outcome

ログイン済みReact、SPA、仮想スクロール、Canvas、独自描画UIには、Semantic locatorだけで安定して操作できない画面がある。一方、汎用click/type/selectを安全操作とみなすと、即時保存や破壊的操作を承認なしで実行する危険がある。

## Evidence and current behavior

- `docs/design.md` §6 はSemantic mode、Visual fallback、site adapterの優先順位を定義している。
- `docs/design.md` §5 はinteractを外部状態への影響で分類する設計を定義している。
- 現在はobserve/interactのポリシーツールが未実装である。

## Reproduction or baseline

テスト用サイトに、通常のARIAボタン、仮想スクロール、Canvas領域、再描画されるSPA要素を用意する。古い要素参照を再利用した操作と、Vision座標操作を検証対象にする。

## Expected behavior

Semantic操作を第一経路とし、取得不能なUIだけVisionへフォールバックする。Visual fallbackは原則承認対象であり、SPA再描画後はsnapshotから対象を再探索する。生のPlaywrightツールは公開しない。

## Scope

- observe: page、links、screenshot
- interact: click、type、select
- Semantic優先の操作ルーティング
- Visionフォールバックの明示状態
- 再snapshotと要素参照の無効化
- 操作ごとのリスク分類

## Non-goals

- 外部状態変更のcommit（Issue #003）
- サイト固有の高水準操作（Issue #005）
- 新しいブラウザエンジン

## Acceptance criteria

- [ ] AC-1: Given an accessible target, when an interaction is requested, then the semantic locator path is selected before visual fallback.
- [ ] AC-2: Given a Canvas or otherwise inaccessible target, when visual fallback is requested, then the action is marked as requiring approval and its coordinate/screenshot context is recorded.
- [ ] AC-3: Given a SPA re-render after scrolling or interaction, when the next action is requested, then the target is re-snapshotted and stale element references are not reused.
- [ ] AC-4: Given generic click/type/select, when the adapter has not classified the operation as safe, then it is not executed as approval-free interact.

## Verification plan

| Criterion | Verification | Expected |
|---|---|---|
| AC-1 | `pnpm test --filter interaction-routing` | semantic path selected |
| AC-2 | `pnpm test --filter interaction-routing` | visual path marked approval-required |
| AC-3 | `pnpm test --filter interaction-routing` | re-snapshot test passes without stale reference |
| AC-4 | `pnpm test --filter interaction-policy` | unclassified generic action is gated |

## Risks, dependencies, rollout

- Dependencies: Issues #001 and #002.
- Security: screenshots may contain sensitive data and obey disclosure policy.
- Rollout: Vision support may remain disabled until screenshot policy is configured.

## Clarifications and assumptions

- Playwright MCP remains the only backend in this phase.
