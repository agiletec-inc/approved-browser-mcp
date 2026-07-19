# integration: AIris MCP Gatewayから承認ブラウザをCOLD起動する

## Problem / outcome

approved-browser-mcpとPlaywright MCPを個別にAIクライアントへ登録すると、生のPlaywrightツールを迂回でき、MCP設定とトークン使用量も分散する。AIris MCP Gatewayから承認層だけを公開し、Playwrightを非公開子プロセスとして起動する必要がある。

## Evidence and current behavior

- `docs/design.md` §3・§4 はGatewayの責務を発見・ライフサイクルに限定し、Playwrightを非公開にする構成を定義している。
- 現在のAIris MCP Gatewayにはこの新規プロバイダー登録は存在しない。

## Reproduction or baseline

実装前のためGatewayからの起動はできない。検証では、Gateway接続後のtools/listにapproved-browser-mcpのポリシーツールだけが見え、Playwrightの生ツールが見えないことを確認する。

## Expected behavior

AIris MCP Gatewayはapproved-browser-mcpをCOLDプロバイダーとして起動し、アイドル停止する。Playwright MCPはapproved-browser-mcpの子プロセスとしてのみ起動し、Gatewayの公開カタログには直接現れない。

## Scope

- Gatewayプロバイダー定義
- host-required capability
- COLD起動・アイドル停止
- tools/listの公開境界
- Claude Code / Claude Desktop接続手順

## Non-goals

- Gatewayへの承認ロジック追加
- Gatewayへのブラウザドライバー追加
- AIris OSの組織管理機能

## Acceptance criteria

- [ ] AC-1: Given a Gateway client, when the browser capability is discovered, then only approved-browser-mcp policy tools are advertised.
- [ ] AC-2: Given the first approved-browser tool call, when the provider is cold, then approved-browser-mcp and its private Playwright child are started successfully.
- [ ] AC-3: Given the idle timeout, when no browser session is active, then both processes stop without leaving a live browser backend.
- [ ] AC-4: Given a direct request for a raw Playwright tool through Gateway, when the request is made, then it is unavailable and cannot bypass policy.

## Verification plan

| Criterion | Verification | Expected |
|---|---|---|
| AC-1 | `pnpm test --filter gateway-integration` | tools/list contains policy tools only |
| AC-2 | `pnpm test --filter gateway-integration` | cold start and child-process health pass |
| AC-3 | `pnpm test --filter gateway-integration` | idle cleanup leaves no backend process |
| AC-4 | `pnpm test --filter gateway-integration` | raw Playwright tool is not routable |

## Risks, dependencies, rollout

- Dependencies: Issues #1, #2, and #3.
- Rollout: register as COLD and keep the existing Gateway behavior unchanged.
- Security: Gateway config must not separately register the Playwright backend.

## Clarifications and assumptions

- Gateway integration is optional for standalone OSS use.
- The first integration targets the existing AIris MCP Gateway repository.
