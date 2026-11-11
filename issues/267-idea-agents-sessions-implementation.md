# Issue #267: Short implementation guide

Source issue: https://github.com/benitogonzalezh/five-panel/issues/267

[Spec](267-idea-agents-sessions.md) · [Technical implementation plan](267-idea-agents-sessions-implementation-plan.md)

Install Paseo, Codex, and our API client once. Configure its Five Panel API address and credential on the instance. The current login token expires after 24 hours and needs replacement separately; messages never rewrite it.

Keep Paseo private through an encrypted SSH connection and a password stored only in Five Panel's backend. Use a dedicated non-root account for this company/user. Five Panel login and API access use HTTPS and the user's existing permissions.

Five Panel adds the chat page, a small SDK adapter, and one conversation-mapping table. **One actions endpoint sends commands; one event stream receives updates.** Display messages, tools, and approvals in one timeline. Paseo stores it, including custom entries for control actions.

## Concrete flow example

Ana opens **Agents** in Zipp. These are proposed Five Panel payloads with sample IDs and an example `product` model.

**1. Start chat.** `POST /agent-sessions` sends `{}`, returning:

```json
{"id":"77777777-7777-4777-8777-777777777777"}
```

The service creates an empty Paseo conversation and saves this mapping projection:

```json
{
  "id":"77777777-7777-4777-8777-777777777777",
  "tenantId":"11111111-1111-4111-8111-111111111111",
  "ownerUserId":"22222222-2222-4222-8222-222222222222",
  "remoteAgentId":"agent_example_01"
}
```

**2. Send an action** to `POST /agent-sessions/77777777-7777-4777-8777-777777777777/actions`:

```json
{"type":"message","data":{"text":"Rename product bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbbb to Travel Mug."}}
```

The service checks ownership, calls SDK `send`, and returns `{"accepted":true}`.

**3. Receive an approval request** through `GET /agent-sessions/77777777-7777-4777-8777-777777777777/events`. The backend maps the SDK request into this `entry` event:

```json
{
  "cursor":{"epoch":"epoch_example","seq":3},
  "entry":{
    "id":"epoch_example:3",
    "direction":"in",
    "type":"permission_request",
    "data":{"requestId":"approval_example_01","tool":"shell","description":"Run five-panel-api to update the product"}
  }
}
```

Ana clicks **Allow**. The same actions endpoint receives:

```json
{"type":"permission_response","data":{"requestId":"approval_example_01","decision":"allow_once"}}
```

The service records the decision in Paseo's timeline, calls SDK `respondToPermission`, and returns `{"accepted":true}`. The installed API credential still decides what Five Panel permits.

**4. Codex calls existing `POST /graphql`** through the API client:

```json
{
  "query":"mutation Rename($id: ID!, $input: updateProductInput!) { updateProduct(id: $id, input: $input) }",
  "variables":{"id":"bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbbb","input":{"name":"Travel Mug"}}
}
```

```json
{"data":{"updateProduct":"bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbbb"}}
```

The data service changes stored `data` from `{"name":"Mug"}` to `{"name":"Travel Mug"}`, with Ana's user ID as `updatedBy`. Normal validation, history, and automations apply.

**5. Reopen the history.** `GET /agent-sessions/77777777-7777-4777-8777-777777777777` returns:

```json
{
  "id":"77777777-7777-4777-8777-777777777777",
  "status":"idle",
  "entries":[
    {"id":"epoch_example:1","direction":"out","type":"message","data":{"text":"Rename product bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbbb to Travel Mug."}},
    {"id":"epoch_example:2","direction":"in","type":"tool_call","data":{"id":"tool_example_01","name":"shell","status":"completed","result":{"data":{"updateProduct":"bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbbb"}}}},
    {"id":"epoch_example:3","direction":"in","type":"permission_request","data":{"requestId":"approval_example_01","tool":"shell","description":"Run five-panel-api to update the product"}},
    {"id":"epoch_example:4","direction":"out","type":"permission_response","data":{"requestId":"approval_example_01","decision":"allow_once"}},
    {"id":"epoch_example:6","direction":"in","type":"message","data":{"text":"Renamed the product to Travel Mug."}}
  ],
  "pendingPermissions":[],
  "nextCursor":null
}
```

The tool result updated entry 2; it did not create a duplicate tool call. Chat shows the whole exchange and the product shows its new name after refresh. Reopening uses `agent_example_01` again. History is displayed; old actions are never executed again.
