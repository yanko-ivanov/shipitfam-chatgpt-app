# Reviewer notes

ShipItFam is an AI dev team you keep on a leash. This app lets ChatGPT drive a user's ShipItFam projects over MCP: list projects, create one from a starter, start missions, show what the crew is waiting on, carry the user's answers back, open previews and ship.

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

- **Projects are created from starters.** `project_starter_list` returns the catalog (key, title, description, category, suggested integrations, first goal) and `project_create` with a `starter_key` makes a local project (branch `main`, no remote repo, `push_mode` off) that can run missions. Only a starter project can run missions. `project_create` without a `starter_key` makes a legacy project that cannot, and the assistant does that only when the user explicitly asks for it. A starter cannot be combined with `repo_url` or a `push_mode` other than `off`.
- **Creating a project provisions a cloud workspace by default**, which takes a few minutes on a subscribed plan. The test cases ask for no workspace (`auto_provision` false), so the review account provisions nothing and costs nothing.
- **No crew runs during the review.** An unprovisioned project has no crew, so a mission created in case P5 stays queued with a "no crew" notice until it is cancelled in the same case. Cases P4 and P5 check the tool calls and the first state, not a finished build. The full loop (create from a starter, start a mission, answer the crew, preview, ship) is in the demo recording.
- **Missions are asynchronous.** On a real project the crew works for minutes to hours and needs the project owner's own Claude login, set up inside the ShipItFam app. A project without it shows "Claude is not logged in on this project" and the assistant points the user to the app.
- **Nothing ships without the user.** Plans, risky commands and "Ship it?" are requests the user answers. The assistant never answers them unless the user says so (case P5 is the explicit exception, and it only adds a note and cancels).
- **Destructive tools** (deletes, stopping or force-resuming a session, rejecting an approval) are annotated destructive and the assistant asks for an explicit yes naming the thing first (case N3).
- **`action` is annotated destructive and open world.** It is the one tool that answers a mission card (approve, answer, comment, ship, cancel), so `destructiveHint` is true on every such call and ChatGPT may show its own confirmation before it runs (case P5). Confirm it; that is expected. It is open world because the "Ship it?" card's Ship action pushes the mission branch to the user's git remote.
- **Billing and Claude logins are not reachable.** Those routes refuse a connected app's token, so the assistant sends the user to the ShipItFam app for them.

## Scope and data

- Every call runs as the signed-in user and sees only that user's projects.
- `openWorldHint` is `true` on exactly two tools, `mcp_server_verify` (calls the URL the user registered) and `action` (the Ship action pushes to the user's git remote), and `false` on the other 76. The full classification of all 78 tools is in `app-metadata.md`.
- Git credentials and MCP-server secrets are write-only and are never returned by any tool. Tool output never includes server addresses or SSH details.
