# browser: Playwright MCPを非公開バックエンドとして接続する

## Problem / outcome

Playwright MCPはブラウザ操作の実装を既に提供している。approved-browser-mcpがブラウザ操作を再実装すると重複と保守負債が発生し、GatewayからPlaywright MCPを直接公開するとポリシー層を迂回できる。

## Evidence and current behavior

- `docs/adr/ADR-0001-approved-policy-layer.md` はPlaywright MCPへの委譲と非公開子プロセス化を決定している。
- `docs/design.md` §3・§4 はapproved-browser-mcpだけをクライアントへ公開する構成を定義している。
- 現在は実装がなく、バックエンド境界は未検証である。

## Reproduction or baseline

実装前のベースラインでは、MCPサーバーもPlaywright接続も存在しない。最小検証は、テスト用ページを対象にPlaywright MCP子プロセスを起動し、approved-browser-mcp経由のobserve呼び出しだけが成功する構成である。

## Expected behavior

approved-browser-mcpはローカルDocker Compose内でPlaywright MCPと専用Chromiumを管理し、クライアントへポリシー層のツールだけを公開する。Playwrightの生ツールや子プロセスのstdin/stdoutはクライアントへ露出しない。ユーザー操作はloopback限定のnoVNC画面へ引き渡す。

## Scope

- MCPサーバーの最小実装
- ローカルDocker Compose内のPlaywright MCP・Chromiumの起動、停止、再接続
- observeリクエストの限定的なプロキシ
- 子プロセスのログ分離と終了処理
- named account用profile volumeとnoVNCのloopback公開
- テスト用ローカルHTMLページ

## Non-goals

- ブラウザドライバーの再実装
- Chrome DevTools MCPやBrowser MCPへの対応
- commit承認フロー
- AIris MCP Gateway統合

## Acceptance criteria

- [ ] AC-1: Given a configured Playwright MCP backend, when approved-browser-mcp starts, then the backend runs as a private child process and only approved-browser-mcp tools are advertised to the client.
- [ ] AC-2: Given a backend tool name not exposed by approved-browser-mcp, when a client requests it, then the request is rejected without forwarding it to Playwright MCP.
- [ ] AC-3: Given the backend exits or emits malformed protocol output, when the client calls a tool, then approved-browser-mcp returns a bounded error and does not hang.
- [ ] AC-4: Given a normal observe request against a local test page, when the request completes, then the extracted result is returned and the child process is terminated cleanly during session stop.
- [ ] AC-5: Given a named account session, when user handoff is requested, then Chromium runs inside the local Docker stack and the interactive screen is available only through a loopback-bound noVNC endpoint.

## Verification plan

| Criterion | Verification | Expected |
|---|---|---|
| AC-1 | `pnpm test --filter backend-boundary` | exit 0; child-process and tool-list tests pass |
| AC-2 | `pnpm test --filter backend-boundary` | exit 0; raw backend tool call is rejected |
| AC-3 | `pnpm test --filter backend-boundary` | exit 0; crash and malformed-output tests pass without timeout |
| AC-4 | `pnpm test --filter backend-boundary` | exit 0; local-page observe and clean-stop tests pass |
| AC-5 | `pnpm test --filter runtime-boundary` | exit 0; container/profile/noVNC boundary tests pass and non-loopback bind is rejected |

## Risks, dependencies, rollout

- Dependency: Playwright MCP and Chromium images must be pinned and resolvable by the local Docker runtime.
- Security: never expose backend credentials or raw protocol output.
- Dependency: this Issue blocks all policy and Gateway work.

## Clarifications and assumptions

- MVP supports Playwright MCP only.
- The backend command and version are configuration, not hard-coded.
