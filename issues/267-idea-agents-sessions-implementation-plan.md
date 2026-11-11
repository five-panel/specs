# Issue #267: Technical implementation plan

Source issue: https://github.com/benitogonzalezh/five-panel/issues/267

[Spec](267-idea-agents-sessions.md) · [Short implementation guide](267-idea-agents-sessions-implementation.md)

Everything below is proposed code. The complete application-source review was performed at `ebccffa359d21ccd7687fc5b2cc3d691ede3919a`. No application or live-instance tests have been run for these documents.

## 1. Connect the existing instance

Configure `PASEO_URL`, `PASEO_PASSWORD`, `PASEO_TENANT_ID`, `PASEO_USER_ID`, and `PASEO_PROVIDER_MODEL`. Only that active company/user can use this trial. Install Paseo, Codex, and our API client and configure private network access manually. Keep the instance's runtime account private to that company/user.

Use a manually configured SSH tunnel to a loopback-only Paseo listener, with a verified SSH host key. The SDK connects to the tunnel's local WebSocket address. Require a strong Paseo password kept in backend settings; the browser never receives it. Run Paseo/Codex under a dedicated non-root account with only this user's files and credentials. Five Panel pages and API calls use HTTPS. The password controls access; SSH encrypts the remote connection. See [Paseo security](https://paseo.sh/docs/security.md).

Add `backend/src/agents/PaseoClient.ts` using `@getpaseo/client`. Pin compatible client/daemon versions and verify create, send, history, custom timeline append, approval response, and stop. Examined SDK 0.15.4 requires `provider/model`; its stop method is low-level `DaemonClient.cancelAgent`. Keep version details inside this adapter. Sources: [SDK](https://paseo.sh/docs/sdk.md), [events](https://paseo.sh/docs/sdk/events.md), [versioned package](https://registry.npmjs.org/@getpaseo/client/0.15.4).

## 2. Use one actions endpoint and one incoming stream

Add `backend/src/db/migrations/038_create_agent_sessions.ts` if that number remains free, and update `backend/src/db/types.ts` for `agent_sessions`: `id`, `tenantId`, `ownerUserId`, nullable `remoteAgentId`, `createdAt`. Paseo holds the timeline; no second history store.

Add `useAgentSessionService` in `backend/src/services/AgentSessionService.ts`, `backend/src/repositories/AgentSessionRepository.ts`, and thin handlers in `backend/src/routes/agentRoutes.ts`. Register through existing `services/context.ts` and `index.ts`. Every route checks current login, configured company/user, and conversation ownership. Foreign IDs return 404.

Revalidate login and membership during long-lived event streams and close them when access expires or is revoked. Keep connection credentials out of browser responses and history. Approval of a tool cannot override normal Five Panel API permissions; an admin API credential grants admin API power.

| Five Panel route | Proposed contract |
| --- | --- |
| `POST /agent-sessions` | `{}` → 201 `{id}`. Create an empty remote conversation and save its mapping. |
| `GET /agent-sessions` | `{items:[{id,createdAt}]}`, owner-only. |
| `GET /agent-sessions/:id` | `{id,status,entries,pendingPermissions,nextCursor}`. Read paged history; optional `before` encodes the native epoch/sequence cursor. |
| `POST /agent-sessions/:id/actions` | `{type,data}` → `{accepted:true}`. Dispatch the three action types below through the existing SDK connection. |
| `GET /agent-sessions/:id/events` | Authenticated fetch/SSE: an initial `snapshot`, then `entry`, `status`, `reset`, or `error` events. Optional `after` encodes the native cursor for reconnect. |

Action types:

- `message`: `data:{text}` → SDK `send(text)`.
- `permission_response`: `data:{requestId,decision:"allow_once"|"deny"}` → SDK `respondToPermission`.
- `stop`: `data:{}` → native cancellation.

Validate each type; unknown types return 400, active-turn text or stale approval returns 409. No separate message/permission/stop routes or arbitrary RPC forwarding. An accepted request means delivery was attempted, not that work finished. On uncertain delivery, refresh history; never automatically resend.

## 3. Keep one typed timeline

Normalize SDK history/events into entries `{id,direction,type,data}`. Direction is `in` from Paseo or `out` from the user. Types cover `message`, `tool_call`, `permission_request`, `permission_response`, `stop`, and `error`. Message data is `{text}`; tool data includes its ID, name, status, and available input/result. Pending permission descriptors contain request ID, tool, and description. Tool results update the existing tool entry.

Paseo already stores messages and tool activity. Record additional approval requests, outgoing decisions, and stop requests as custom timeline items with SDK `timeline.append`. Use `kind:"five_panel.control"`, `version:1`, a stable item ID, and data `{direction,type,data}`. These are history records; the real action still calls its SDK method. Record requested actions as requests, and confirm their outcome only from SDK state. Never dispatch an action while replaying history.

A concrete append input for the guide's approval decision is:

```json
{
  "type":"plugin",
  "id":"fp_permission_response_approval_example_01",
  "kind":"five_panel.control",
  "version":1,
  "data":{
    "direction":"out",
    "type":"permission_response",
    "data":{"requestId":"approval_example_01","decision":"allow_once"}
  }
}
```

Derive entry IDs from native item identity or epoch/sequence start; retain that ID as tool/text output grows. Do not duplicate native user messages. Check before appending the same control item again. Verify append/refetch persistence against the installed daemon; require a compatible release rather than quietly losing control history.

The SSE `snapshot` data matches the history response. An `entry` event carries `{cursor,entry}`; `status` carries `{status}`. Upsert entries by ID. Subscribe and await readiness before dispatch. On reconnect, refetch missed native history by epoch/sequence and deduplicate; a changed epoch sends `reset`. Use explicit turn completion/failure/cancellation, not an idle transition alone. Release observations on page close. No queue, supervisor, or event mirror.

## 4. Reuse existing API access and build the page

Add a small `five-panel-api` client and endpoint reference under `ops/agents/`. Support `describe` for existing APIs/live model fields and `request` for REST/GraphQL. During manual setup, save the API URL and the configured user's ordinary login token in its private configuration file. Existing `POST /auth/login` supplies the token; no new authorization endpoint. Current tokens expire after 24 hours; replace them through the existing login flow when needed. Chat never rewrites configuration. Keep credentials out of prompts, responses, URLs, and logs.

Existing `middleware/auth.ts`, `services/SessionService.ts`, `schema.ts`, `ModelService.ts`, and `DataService.ts` retain current permissions, validation, history, and automations. No database access or task-specific capability list.

Add `views/AgentSessionsView.vue`, `services/AgentSessionService.ts`, and `composables/useAgentSession.ts` under `frontend/src`; extend the router/sidebar using MainLayout and ApiService. Show one timeline, typed action controls, and connection status. Use authenticated fetch for SSE; clear private state/streams on logout, company change, and unmount. Render safe text, mask Clarity recordings, and add English/Spanish labels.

Verify both-direction history after reopen/daemon restart; tool-entry updates; approval/stop recording; stale approvals; unknown types; no duplicate/replayed actions after reconnect; owner/company rejection; wrong Paseo password; private listener access; stream closure after revoked/expired login; unchanged credentials; and a real Codex API write with normal history. Run backend/frontend tests/builds and a focused browser journey after implementation. Keep imports, reports, files, and automations working. Use the manually managed instance's existing backup.
