<div align="center">

<img src="https://cdn.jsdelivr.net/npm/@communicate_so/cli@0.1.5/assets/communicate-icon.png" alt="Communicate" width="128" height="128" />

# Communicate CLI

**Manage your AI support operations from the terminal.**

Scoped credentials · Named commands · Structured JSON · OpenAPI contracts

[![npm version](https://img.shields.io/npm/v/%40communicate_so%2Fcli?style=flat-square&color=0f766e)](https://www.npmjs.com/package/@communicate_so/cli)
[![Node.js](https://img.shields.io/badge/Node.js-22%2B-333333?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![OpenAPI](https://img.shields.io/badge/OpenAPI-3.2.1-0f766e?style=flat-square)](https://app.communicate.so/api/v1/openapi.json)

[Website](https://communicate.so) · [Documentation](https://developer.communicate.so) · [API contract](https://app.communicate.so/api/v1/openapi.json)

</div>

---

[Quick start](#quick-start) · [Authentication](#authentication) · [Workflows](#workflows) · [Automation](#automation) · [Security](#security) · [Development](#development)

## Quick start

Requires **Node.js 22 or later**.

```sh
npm install --global @communicate_so/cli
communicate --version
communicate --help
```

Inspect a command before running it. Help includes its validated input schema.

```sh
communicate agents create --help
communicate sources upload --help
```

## Authentication

Use an existing scoped credential from your secret manager through `COMMUNICATE_TOKEN`, or a protected credential file:

```sh
communicate workspaces list --token-file credential.json
```

Credential files must be owned by you, regular files with **mode 0600**, and not symlinks. Explicit files take precedence over the environment.

### Create a credential without a browser

Verified password users can bootstrap an account credential. Inspect the schema, then supply email, password, name and requested scopes through stdin or a protected input file.

```sh
communicate auth credentials bootstrap --help
communicate auth credentials bootstrap --input - --output-file credential.json < proof.json
communicate workspaces list --token-file credential.json
```

The output file is created exclusively with mode 0600. An existing file is never overwritten. Store proof files securely and remove them when no longer needed.

> Request only the scopes your workflow needs. Credential creation, editing, rotation and revocation require current password proof; creation and rotation also require a protected output destination.

## Workflows

This increment provides named commands for these management workflows. Run `communicate --help` for the authoritative command list and availability.

| Workflow | Command family |
| :--- | :--- |
| Create, inspect and configure workspaces | `workspaces` |
| Manage workspace membership and invitations | `members` |
| Organize teams and their members | `teams` |
| Create and configure support agents | `agents` |
| Add, upload, inspect, edit and refresh knowledge sources | `sources` |
| Review learning and run Grill workflows | `learnings`, `grill` |
| Configure personal notifications and preferences | `notifications` |
| Configure custom actions and atomic batches | `actions` |
| Connect and manage scheduling integrations | `integrations` |
| Create, edit, duplicate, publish, pause and archive in-app campaigns | `in-app-campaigns` |
| Prepare and confirm agent-owned image uploads | `in-app-images` |
| Create, inspect, pause and delete outbound campaign drafts | `outbound-campaigns` |
| Inspect ticket threads, manage status, queue replies, undo and edit sent widget replies | `inbox` |
| Manage scoped account credentials | `auth credentials` |

### Create a workspace and agent

Pass structured settings through JSON files or stdin. Use command help to see required fields.

```sh
communicate workspaces create --input workspace.json --token-file credential.json
communicate agents create --workspace WORKSPACE_ID --input agent.json --token-file credential.json
```

### Add knowledge

```sh
communicate sources upload --workspace WORKSPACE_ID --agent AGENT_ID --file knowledge.txt --token-file credential.json
communicate sources list --workspace WORKSPACE_ID --agent AGENT_ID --token-file credential.json
```

### Review and manage tickets

Request `inbox:read` for lists, counts and full threads, and `inbox:write` for replies, ticket changes and explicit read markers. Existing credentials gain no additional authority. Thread reads do not mark customer messages as seen.

```sh
communicate inbox list --workspace WORKSPACE_ID --token-file credential.json
communicate inbox counts --workspace WORKSPACE_ID --token-file credential.json
communicate inbox get --workspace WORKSPACE_ID --input thread.json --token-file credential.json
communicate inbox mark-read --workspace WORKSPACE_ID --input thread.json --token-file credential.json
communicate inbox reply --workspace WORKSPACE_ID --input reply.json --token-file credential.json
communicate inbox update --workspace WORKSPACE_ID --input ticket-update.json --token-file credential.json
```

`thread.json` contains `conversationId`. `inbox mark-read` advances only your member read marker through the latest committed message or mention and clears your pending notifications; other members are unchanged. A reply also requires `content` and a stable UUID `messageId`; retain the same ID when reconciling a repeated submission. Acceptance means queued, not delivered. `inbox replies undo` uses `conversationId` and `messageId` before the undo deadline. `inbox replies edit` requires `content` and the current `expectedVersion`; only sent widget replies within their edit window can be edited.

Ticket updates support status, assignment, team, priority, due date, handling mode and contact email. Closing a ticket with live support requests requires explicit, versioned dispositions; an unresolved customer request is not silently marked resolved. `notify` controls the optional customer closure email. Inspect command help for the complete schema before writing.

### Preview a request

`--dry-run` validates input and shows the planned method and route without sending a request.

```sh
communicate agents create --workspace WORKSPACE_ID --input agent.json --dry-run
```

> **Release scope:** complete product parity, billing workflows, automatic pagination, profiles, credential renewal, job/event workflows and MCP writes are still being delivered incrementally. This CLI does not claim every Communicate workflow is available yet.

## MCP read and write

Connect MCP clients to `https://app.communicate.so/mcp`. Management tools require an explicitly MCP-bound account credential: include `"audience": "https://app.communicate.so/mcp"` and the required scopes in the credential bootstrap input. Supply the returned token through your client's protected secret configuration.

REST credentials are isolated from MCP credentials. Existing workspace OAuth tokens retain their read-only agent-list access. Account management tools support SDK v1 and SDK v2 clients, including protocol revisions `2025-11-25` and `2026-07-28`.

MCP document uploads accept a bounded filename and base64 file bytes; the server does not read local filesystem paths. Provider consent, device delivery, and billing or inbox workflows beyond the registered manifest remain separate requirements. The platform coverage is still partial.

## Automation

Success writes JSON to stdout:

```json
{ "ok": true, "data": {} }
```

Array results can be emitted as newline-delimited JSON:

```sh
communicate workspaces list --format ndjson --token-file credential.json
```

Failures write structured JSON to stderr. Use exit codes to decide how your automation responds.

| Exit | Meaning |
| ---: | :--- |
| `0` | Success |
| `2` | Input or configuration error |
| `3` | Authentication or authorization failure |
| `4` | Conflict |
| `5` | Quota or rate limit |
| `6` | Local or transport failure |
| `7` | API or response failure |

The default API URL is `https://app.communicate.so/api/v1`. Override it with `COMMUNICATE_API_URL` or `--base-url`. `--timeout-ms` accepts 1–300000 milliseconds.

## Security

- **Keep secrets out of arguments.** Supply passwords and credentials through protected files, stdin or your secret manager.
- **Protect one-time results.** `--output-file` stores the complete response, including secrets; ordinary stdout redacts secret and invitation-link fields.
- **Use secure transport.** HTTPS is required except for HTTP on loopback. Authenticated redirects are refused.
- **Reconcile before retrying writes.** Requests are not automatically retried. A transport failure can leave the server outcome uncertain.
- **Control credential storage.** The CLI does not load dotenv files or persist credentials automatically. Protect output files containing customer data.

Report sensitive security issues privately through [Communicate](https://communicate.so), rather than posting credentials or customer data in a public issue.

## Development

From the Communicate platform repository:

```sh
pnpm build:cli
pnpm test:cli-install
```

The distribution check builds, packs and installs outside the monorepo, verifies every named command and checks that installed JavaScript matches the tested artifact.

---

<div align="center">

**Communicate** · AI support operations, accessible to your tools.

[Get started](https://communicate.so) · [Read the docs](https://developer.communicate.so)

</div>
