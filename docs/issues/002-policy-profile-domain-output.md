# security: 名前付きプロファイルと公開境界を実装する

## Problem / outcome

ログイン済みページの本文やスクリーンショットはAIプロバイダーへ送信され得る。個人ブラウザの誤接続、危険URL、許可外ドメイン、Cookieや認証ヘッダーの漏出を防ぐP0境界が必要である。

## Evidence and current behavior

- `docs/design.md` §6–§8 はアカウントID、プロファイル、ドメイン、公開レベルをクライアント単位で制御する設計を定義している。
- `docs/adr/ADR-0001-approved-policy-layer.md` は名前付きアカウントと実ログインアカウントの照合を決定している。
- 現在は設定、プロファイルロック、出力フィルターが未実装である。

## Reproduction or baseline

実装前のため、許可ドメインや公開レベルを評価するコードはない。テストでは、許可外URL、`file://`、localhost、LAN内IP、クラウドメタデータ宛先、Cookieを含む出力を最小入力として扱う。

## Expected behavior

クライアントは名前付きプロファイルと許可ドメインに束縛される。既定ではmetadata/textのみを返し、危険URL・許可外リダイレクト・機密ヘッダーは拒否またはマスクする。同じプロファイルは排他ロックされる。

## Scope

- クライアント、アカウントID、プロファイル、ドメイン許可設定
- 危険URLとリダイレクト検査
- metadata/text/screenshot/sensitive公開レベル
- Cookie、Authorization、パスワード、2FA、決済情報の出力除外
- プロファイル単位の排他ロック

## Non-goals

- 組織SSO、RBAC、端末管理（AIris OS）
- ブラウザ起動実装（Issue #001）
- サイト固有のログインアカウント検証（後続Issue）

## Acceptance criteria

- [ ] AC-1: Given a client profile with an allowlist, when navigation targets an unlisted domain or redirect, then the request is rejected before reaching the backend.
- [ ] AC-2: Given a URL using `file://`, `chrome://`, localhost, a loopback/LAN address, or a cloud metadata address, when navigation is requested, then the request is rejected by default.
- [ ] AC-3: Given backend output containing cookies, Authorization headers, passwords, 2FA codes, or payment fields, when the result is returned or logged, then those values are absent or redacted.
- [ ] AC-4: Given two sessions request the same named profile concurrently, when the second session starts, then it is rejected without modifying the first session.
- [ ] AC-5: Given the default policy, when a page is read, then metadata and text may be returned while screenshot and sensitive levels require explicit configuration.

## Verification plan

| Criterion | Verification | Expected |
|---|---|---|
| AC-1 | `pnpm test --filter policy-boundary` | allowlist and redirect tests pass |
| AC-2 | `pnpm test --filter policy-boundary` | dangerous URL tests pass |
| AC-3 | `pnpm test --filter output-filter` | secret redaction tests pass with no raw values |
| AC-4 | `pnpm test --filter profile-lock` | concurrent lock test rejects the second session |
| AC-5 | `pnpm test --filter disclosure-policy` | default disclosure matrix passes |

## Risks, dependencies, rollout

- Dependency: Issue #1.
- Security: fail closed when configuration is missing or malformed.
- Data: do not persist page bodies by default.

## Clarifications and assumptions

- Profile credentials remain in the Playwright-managed browser profile; this layer never reads raw cookies.
- `sensitive` is an explicit user/configuration permission, not an automatic classifier claim.
