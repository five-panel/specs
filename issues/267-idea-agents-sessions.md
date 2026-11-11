# Issue #267: Idea: Agents sessions

Source issue: https://github.com/benitogonzalezh/five-panel/issues/267

## TL;DR

Chat with Codex from Five Panel. Codex runs on your existing instance through Paseo and uses Five Panel's data and actions through its existing API, with your permissions.

[Technical implementation plan](267-idea-agents-sessions-implementation-plan.md) · [Short implementation guide](267-idea-agents-sessions-implementation.md)

## What will change

**Medium code impact:** add a chat page, a small Paseo SDK connection service, one table linking conversations to their owner, and an API client for Codex. Reuse existing login, tenant checks, REST/GraphQL APIs, validation, and record history.

## How it would work

1. Install Paseo and Codex on your instance and sign in. Configure the Paseo connection in Five Panel and the existing Five Panel API address and credential on the instance once. Replace the credential only when it expires or changes. The first trial uses a trusted instance for the configured company/user.
2. Open **Agents** and start a conversation. Use one outgoing actions endpoint for text, approval decisions, and stop requests. One incoming event stream carries replies, tool activity, approval requests, and status changes.
3. Show these as typed entries in one conversation timeline. For example: your message → a tool call → an approval request → your answer → the tool result → Codex's reply. The SDK handles communication with Codex.
4. Codex uses a small client for the existing Five Panel APIs and live model fields. It can read/change data, manage models, and use other actions allowed to you. Keep history in Paseo, including control actions, and save its conversation ID in Five Panel. Reopening loads history without executing old actions again.
5. Keep Paseo private behind an encrypted SSH connection and a password held only by the backend. Use HTTPS and existing login for Five Panel and its API. Run the instance under a dedicated non-root account for this company/user; sharing a Paseo password does not isolate companies.

## Done when

- Send messages, see replies and tool activity, answer approvals, and stop a turn through the same actions path.
- Reopening shows messages and actions in both directions, including approval decisions and stop requests.
- Ask Codex to change a record; the change appears with normal validation and history.
- Disconnecting shows a clear state and reconnecting fills missed history without repeating instructions or actions.
- API calls follow your current permissions; another company/user cannot open your conversations. Connection authentication is required, revoked access closes the stream, and messages leave credential configuration unchanged.

## Out of scope

- Automatic machine setup, host management, shared-host isolation, and capacity scheduling.
- Scheduled agents, record-specific launch buttons, special-purpose agents, and new business rules.
- Attachments and remote-desktop screens. Other providers can follow later through Paseo.
