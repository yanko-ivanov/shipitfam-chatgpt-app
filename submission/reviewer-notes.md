# Reviewer notes

ShipItFam is an AI dev team you keep on a leash. This app lets ChatGPT run a user's ShipItFam projects over MCP: list projects, create one from a starter, queue missions, show what the crew is waiting on, carry the user's approvals and answers back, follow up on finished work, set how much the crew asks, open previews and ship.

- MCP server: `https://shipitfam.com/mcp` (Streamable HTTP)
- Website: https://shipitfam.com
- Support: https://shipitfam.com/contact.html

## Test account

TODO(owner): provide the reviewer test account here before submitting.

- Email: TODO
- Password: TODO
- Multi-factor authentication: none. The account signs in with email and password only.
- Plan: Free. A starter project is refused on a plan whose workspace is a pod (`409 starter_unavailable_on_substrate`), and the cases below need to create one.
- Seeded data: none required. Every positive case is read-only or creates its own fresh state (a project, a mission), so the cases can be run again and in any order, by any number of reviewers. Optionally keep one starter project on the account so cases P1 and P2 have something to show; they are written to pass on an empty account too.
- Cleanup: P4 and P5 each leave one small unprovisioned project named "ShipItFam review ..." behind. They cost nothing; the owner deletes them now and then.

Do not commit the real credentials to this public repository. Enter them in the OpenAI dashboard's reviewer-notes field instead.

## OAuth facts

| Item | Value |
|---|---|
| Flow | Authorization code with PKCE. `code_challenge_method` must be `S256`. |
| Client type | Public. `token_endpoint_auth_methods_supported` is `["none"]`. No client secret. |
| Client registration | Dynamic client registration (RFC 7591) at `https://shipitfam.com/api/oauth/register`. |
| Redirect URI for ChatGPT | `https://chatgpt.com/connector_platform_oauth_redirect`. Redirect URIs must be `https`, or `http` on a loopback host. Matching is exact; loopback hosts ignore the port. |
| Issuer | `https://shipitfam.com` |
| Protected resource metadata | `https://shipitfam.com/.well-known/oauth-protected-resource/mcp` (also served at `/.well-known/oauth-protected-resource`) |
| Authorization server metadata | `https://shipitfam.com/.well-known/oauth-authorization-server` (same body at `/.well-known/openid-configuration`) |
| Authorization endpoint | `https://shipitfam.com/api/oauth/authorize` |
| Token endpoint | `https://shipitfam.com/api/oauth/token` (form-encoded or JSON) |
| Scope | `mcp` |
| Resource indicator | RFC 8707 `resource` is accepted and must equal `https://shipitfam.com/mcp`. |
| Authorization response | The redirect always carries `iss=https://shipitfam.com` (RFC 9207). The metadata advertises `authorization_response_iss_parameter_supported: true`. |
| Authorization code | Single use, expires after 60 seconds. |
| Access token | An MCP access token, valid 24 hours. |
| Refresh token | Rotating: each refresh issues a new refresh token and the old one stops working. |
| Revocation | By the user, under MCP access in ShipItFam. A revoked connection fails its next refresh with `invalid_grant`. |
| Unauthenticated `/mcp` | Responds `401` with `WWW-Authenticate: Bearer resource_metadata="https://shipitfam.com/.well-known/oauth-protected-resource/mcp"`. |

### What the reviewer sees

1. Add ShipItFam in ChatGPT and click Connect.
2. ChatGPT registers itself, then opens ShipItFam's consent page in the browser.
3. Log in with the test account (email and password, no MFA) on the consent page.
4. The page says ChatGPT wants to access the ShipItFam account and shows the return address. Click **Allow**. **Deny** returns `access_denied`.
5. The browser returns to ChatGPT and the tools are available.

## Behavior to expect

- **Projects are created from starters.** `project_starter_list` returns the catalog (key, title, description, category, suggested integrations, first goal) and `project_create` with a `starter_key` makes a local project (branch `main`, no remote repo, `push_mode` off) that can run missions. A project runs missions when its `core_v2` is true, and a new project gets that from a starter. `project_create` without a `starter_key` makes a legacy project (`core_v2` false) that cannot, and the assistant does that only when the user explicitly asks for it. A starter cannot be combined with `repo_url` or a `push_mode` other than `off`.
- **Creating a project provisions a cloud workspace by default**, which takes a few minutes on a subscribed plan. The test cases ask for no workspace (`auto_provision` false), so the review account provisions nothing and costs nothing.
- **Missions are a queue.** `mission_create` puts a mission at the end of the project's queue. They run oldest first, one at a time per crew, and cannot be reordered, only cancelled. `mission_list` shows the queue (Needs you, In progress, Up next, Done), `mission_get` one mission with its steps, `job_log` what the crew is doing and `step_diff` the code a step changed.
- **No crew runs during the review.** An unprovisioned project has no crew, so a mission created in case P5 stays in Up next with a "no crew" notice (the `mission_create` answer carries it) until it is cancelled in the same case. Cases P4 and P5 check the tool calls and the first state, not a finished build. The full loop (create from a starter, queue a mission, approve the plan, answer a question, watch the log, set the settings, follow up, preview, ship) is in the demo recording.
- **Missions are asynchronous.** On a real project the crew works for minutes to hours and needs the project owner's own Claude login, set up inside the ShipItFam app. A project without it shows "Claude is not logged in on this project": the assistant offers to reuse a login from another of the user's projects (`request_use_login`) or points the user to the app.
- **Nothing is decided for the user.** Plans, risky commands, step reviews, questions, failures and "Ship it?" are requests the user answers with `request_answer`. The assistant reads the request in full (`request_get`), explains it, and asks before it approves a plan, allows a command, ships, skips or cancels, unless the user already said to ("approve the plan"). It never invents an answer to the crew's question.
- **Follow-ups and settings.** `mission_comment` reopens a finished mission with follow-up work, steers a running step, or leaves a note; it never answers a request. `project_settings_set` sets, per project, whether the crew stops for plan approval, asks before risky commands, pauses after every step, and keeps working on the next mission while one waits. The assistant says what a change does before it makes one and does not loosen a setting on its own.
- **Destructive tools** (deletes, `mission_cancel`, revoking an invite or a git credential) are annotated destructive and the assistant asks for an explicit yes naming the thing first (case N3).
- **`request_answer` is annotated destructive and open world.** It is the one tool that answers a request (approve, reply, skip, ship), so `destructiveHint` is true on every such call and ChatGPT may show its own confirmation before it runs. Confirm it; that is expected. It is open world because the "Ship it?" request's approve option pushes the mission branch to the user's git remote.
- **Connected apps.** `integration_list` shows which apps (Slack, HubSpot and the like) a project can connect and which it has. On a server where connecting apps is not set up it says so (`nango` is "not_configured" or "unreachable") and the assistant relays that and stops. `integration_connect` takes two calls: the first returns a `connect_link` that the user must open in their own browser and authorize, and the app is not connected until they have; the second, with `finished` true, records it. Only a lead of the project can connect or disconnect an app, and an app connects read-only unless the user says the crew may write through it. `integration_disconnect` is annotated destructive. None of the three reaches outside ShipItFam itself, so they are not open world.
- **Skill import is reviewed by the user, never by the assistant.** `agent_type_marketplace_search` and `agent_type_import_preview` fetch from SkillsMP and GitHub (open world, read-only, nothing stored). The preview returns the whole skill text, its licence and the analyzer findings, and the assistant shows all of it to the user before it calls `agent_type_import`. A skill is third-party prompt text, so ShipItFam refuses any finding or licence acknowledgement that arrives through a connected app (`403 app_review_required`): only a skill with no high-severity findings and a permissive licence is importable from here, and any other is imported in the ShipItFam app. `agent_type_import_check` and `agent_type_import_update` do the same for an update of an imported role.
- **Billing and Claude logins are not reachable.** Those routes refuse a connected app's token, so the assistant sends the user to the ShipItFam app for them.
- **What the cases cannot exercise.** `request_get`, `request_answer`, `request_use_login`, `job_log`, `step_diff`, `preview_pick` and `project_wake` need a mission that a running crew has taken as far as a request, a step or a preview, and `integration_connect`, `integration_disconnect`, `agent_type_import` and `agent_type_import_update` need the user's own browser or a reviewed skill. See "Not covered by these cases" in `test-cases.md`.

## Scope and data

- Every call runs as the signed-in user and sees only that user's projects.
- `openWorldHint` is `true` on exactly seven tools: `mcp_server_verify` (calls the URL the user registered), `request_answer` (the Ship option pushes to the user's git remote) and the five skill marketplace tools `agent_type_marketplace_search`, `agent_type_import_preview`, `agent_type_import`, `agent_type_import_check` and `agent_type_import_update` (they fetch from SkillsMP and GitHub). It is `false` on the other 58. The full classification of all 65 tools is in `app-metadata.md`.
- Git credentials and MCP-server secrets are write-only and are never returned by any tool. Tool output never includes server addresses or SSH details.
