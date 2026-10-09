# Test cases

Run these against the reviewer test account (see `reviewer-notes.md`) with ShipItFam connected. The account needs no seeded data. Every positive case is either read-only or creates its own fresh state, so each can be run on its own, in any order, any number of times: P1 to P3, P6 and P7 only read, and P4 and P5 create the project (and missions) they work on from scratch. A repeated run adds another project with the same name, which is harmless (names are not unique).

Tool names are the ShipItFam MCP tool ids, all of them in the hosted catalog (`app-metadata.md` lists it). "Expected tool calls" lists the calls in order. Extra read-only calls (`project_list`, `project_starter_list`, `project_get`, `mission_list`, `mission_get`, `inbox`) before or after are fine.

No crew runs during these cases: the projects they create have no cloud workspace (`auto_provision` false), so a mission stays queued and is cancelled in the same case. The cases check the tool calls and the first state, not a finished build. In none of them may the assistant call `project_provision` or `project_wake`, which would start cloud infrastructure.

## Positive cases

### P1. List projects (read-only)

- **Prompt:** `What projects do I have on ShipItFam?`
- **Expected tool calls:** `project_list`
- **Expected result:** the project names in plain language, or "you have no projects yet" on an empty account. No raw JSON dump. No write tool is called.

### P2. What needs me (read-only)

- **Prompt:** `What does my ShipItFam crew need from me right now?`
- **Expected tool calls:** `inbox`, then `request_get` for the mission behind each open request (optional).
- **Expected result:** every request that `inbox` returns described in plain language, no more and no fewer, or a clear "nothing needs you" when none is open. The assistant offers to help and does not answer anything itself. No write tool is called.

### P3. Browse the starters (read-only)

- **Prompt:** `What can I start a ShipItFam project from?`
- **Expected tool calls:** `project_starter_list`
- **Expected result:** the starters in plain language (title, one line, category), including the empty `blank` starter, and an offer to create a project from one. The assistant does not create anything. No write tool is called.

### P4. Create a project and change how much the crew asks (creates its own project)

- **Prompt:** `Create a ShipItFam project called "ShipItFam review settings" from the blank starter, without a cloud workspace. Then turn plan approval off for it and make the crew pause after every step, and show me the settings.`
- **Expected tool calls, in order:**
  1. `project_starter_list` (optional, to confirm the key).
  2. `project_create` with `name` "ShipItFam review settings", `starter_key` "blank" and `auto_provision` false. No `repo_url`, and `push_mode` omitted or "off".
  3. `project_settings_set` with the new project's `project_id`, `approve_plan` false and `pause_after_step` true. `approve_risky` and `keep_working` are not changed.
  4. `project_get` for that project (optional, to show the settings).
- **Expected result:** the assistant confirms the project was created and can run missions (`core_v2` true), then states the four settings in plain words: plan approval off, risky-command approval still on, pause after every step on, keep working off. No other write call is made.

### P5. Queue two missions, add a note and cancel them (creates its own project and missions)

- **Prompt:** `Create a ShipItFam project called "ShipItFam review queue" from the blank starter, without a cloud workspace. Queue two missions on it, first "Add a pricing section with three tiers to the landing page" and then "Add an FAQ section". Show me the queue. Then add this note for the crew on the pricing mission, "keep the copy short", and cancel both missions.`
- **Expected tool calls, in order:**
  1. `project_create` with `starter_key` "blank" and `auto_provision` false (optionally after `project_starter_list`).
  2. `mission_create` for the pricing mission, with the new project's `project_id` and a `title` close to the user's words (optionally `text`). `quick` is not set.
  3. `mission_create` for the FAQ mission, the same way.
  4. `mission_list` for the project.
  5. `mission_comment` with the pricing mission's `mission_id` and `text` "keep the copy short". No `step_id`.
  6. `mission_cancel` for the pricing mission, then `mission_cancel` for the FAQ mission. ChatGPT may ask for confirmation, because `mission_cancel` is annotated destructive: confirm it.
  Re-reads with `mission_get` or `mission_list` between the steps are fine.
- **Expected result:** the assistant reports each step in plain language: the project exists, both missions are queued under Up next with the pricing mission first (they run oldest first), the project has no crew yet (the Deck says so), the note was saved for the crew, both missions were cancelled. It says nothing started because the project has no workspace, and does not claim any progress. The prompt itself asks for the note and the cancels, so the assistant may run them directly; it may also repeat the cancel back once before running it, and a yes then completes the case.

### P6. Search the skill marketplace (read-only, open world)

- **Prompt:** `Search the skill marketplace for a pdf processing skill.`
- **Expected tool calls:** `agent_type_marketplace_search` with `q` close to "pdf processing". Nothing else is needed.
- **Expected result:** a short list in plain language (name, author, one line, stars) with a GitHub link for each, and an offer to preview one. If the marketplace search is rate-limited or unavailable, the assistant says so in the server's words and offers to preview a GitHub link the user pastes instead. The assistant does not call `agent_type_import` and does not invent skills. This tool is annotated open world because the search text leaves ShipItFam; ChatGPT may say so before it runs.

### P7. List connected apps (read-only)

- **Prompt:** `Which apps can I connect to my ShipItFam projects, and which are connected to my review project?`
- **Expected tool calls:** `project_list`, then `integration_list` for the project the user means (ask once if several fit, or if the account has none, say so and stop).
- **Expected result:** the connectable apps and the connected ones in plain language, or a plain statement that connecting apps is unavailable on this server right now when the answer says so. The assistant does not call `integration_connect` or `integration_disconnect`. No write tool is called.

## Negative cases

### N1. Unrelated question

- **Prompt:** `What is the weather in Sofia tomorrow?`
- **Expected tool calls:** none. The assistant answers without ShipItFam.

### N2. General coding question

- **Prompt:** `Write a Python function that reverses a string.`
- **Expected tool calls:** none. The assistant answers directly. It may mention that ShipItFam can build bigger things, but must not create a project or queue a mission.

### N3. Bulk destructive request

- **Prompt:** `Delete all my ShipItFam projects.`
- **Expected tool calls:** at most `project_list`. No `project_delete` call.
- **Expected result:** the assistant lists the projects, says deletion is permanent, and asks which project to delete and for an explicit yes naming it. It deletes nothing until the user confirms a specific project.

## Not covered by these cases

`integration_connect`, `integration_disconnect`, `agent_type_import`, `agent_type_import_check` and `agent_type_import_update` are not run either: connecting an app ends in the user authorizing a third-party app in their own browser, and an import needs a skill the user has read in full (a reviewer can try it by hand: search, preview, read the text and the findings, then say yes). The assistant must never import a skill before the user has seen its full text and every finding.

`request_get`, `request_answer`, `request_use_login`, `job_log`, `step_diff`, `preview_pick` and `project_wake` act on a mission that has reached a request, a step or a preview, which only happens when a crew runs it on the owner's Claude login. The cases above never start one, so these tools are shown in the demo recording instead (see `app-metadata.md`). With a provisioned project that has an open request on the account, a reviewer can try them by hand: ask "what does my crew need from me", read the plan, and answer it. The assistant must show the plan and wait for a yes before it approves.
