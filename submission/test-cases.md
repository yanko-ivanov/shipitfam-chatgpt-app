# Test cases

Run these against the reviewer test account (see `reviewer-notes.md`) with ShipItFam connected. The account needs no seeded data. Every positive case is either read-only or creates its own fresh state, so each can be run on its own, in any order, any number of times: P1 to P3 only read, and P4 and P5 create the project (and mission) they work on from scratch. A repeated run adds another project with the same name, which is harmless (names are not unique).

Tool names are the ShipItFam MCP tool ids. "Expected tool calls" lists the calls in order. Extra read-only calls (`project_list`, `project_starter_list`, `mission_get`, `helm_feed`) before or after are fine.

No crew runs during these cases: the projects they create have no cloud workspace (`auto_provision` false), so a mission created in P5 stays queued and is cancelled in the same case. The cases check the tool calls and the first state, not a finished build.

## Positive cases

### P1. List projects (read-only)

- **Prompt:** `What projects do I have on ShipItFam?`
- **Expected tool calls:** `project_list`
- **Expected result:** the project names in plain language, or "you have no projects yet" on an empty account. No raw JSON dump. No write tool is called.

### P2. What needs me (read-only)

- **Prompt:** `What does my ShipItFam crew need from me right now?`
- **Expected tool calls:** `request_list`, then `mission_get` for the mission behind each open request (optional). On an account with a legacy project the assistant also calls `helm_feed` before it says nothing is waiting.
- **Expected result:** every request that `request_list` returns described in plain language, no more and no fewer, or a clear "nothing needs you" when none is open. The assistant offers to help and does not answer anything itself. No write tool is called.

### P3. Browse the starters (read-only)

- **Prompt:** `What can I start a ShipItFam project from?`
- **Expected tool calls:** `project_starter_list`
- **Expected result:** the starters in plain language (title, one line, category), including the empty `blank` starter, and an offer to create a project from one. The assistant does not create anything. No write tool is called.

### P4. Create a project from a starter (creates its own project)

- **Prompt:** `Create a ShipItFam project called "ShipItFam review check" from the blank starter. Don't set up a cloud workspace yet.`
- **Expected tool calls:** `project_starter_list` (optional, to confirm the key), then `project_create` with `name` "ShipItFam review check", `starter_key` "blank" and `auto_provision` false. No `repo_url`, and `push_mode` omitted or "off".
- **Expected result:** the assistant confirms the project was created, that it can run missions (`core_v2` true), that it has no remote repo and no cloud workspace yet, and offers to start a mission on it. No other write call is made.

### P5. Start, steer and cancel a mission (creates its own project and mission)

- **Prompt:** `Create a ShipItFam project called "ShipItFam review mission" from the blank starter, without a cloud workspace. Start a mission on it: add a pricing section with three tiers to the landing page. Then add this note for the crew, "keep the copy short", and cancel the mission.`
- **Expected tool calls, in order:**
  1. `project_create` with `starter_key` "blank" and `auto_provision` false (optionally after `project_starter_list`).
  2. `mission_create` with the new project's `project_id` and a `title` close to the user's words (optionally `text`). `quick` is not set.
  3. `mission_get` for the new mission, to read the actions its card offers.
  4. `action` with that `mission_id`, `action_id` "comment" and `text` "keep the copy short".
  5. `action` with that `mission_id` and the cancel id the card lists (`cancel`, or `skip` labelled "Cancel mission" when the open request offers it). ChatGPT may ask for confirmation, because `action` is annotated destructive: confirm it.
  Re-reads with `mission_get` between the steps are fine.
- **Expected result:** the assistant reports each step in plain language: the project exists, the mission was created and what the crew would do first (write a spec and maybe ask questions), the note was added, the mission was cancelled. It says no crew ran because the project has no workspace, and does not claim any progress. The prompt itself asks for the note and the cancel, so the assistant may run them directly; it may also repeat the cancel back once before running it, and a yes then completes the case.

## Negative cases

### N1. Unrelated question

- **Prompt:** `What is the weather in Sofia tomorrow?`
- **Expected tool calls:** none. The assistant answers without ShipItFam.

### N2. General coding question

- **Prompt:** `Write a Python function that reverses a string.`
- **Expected tool calls:** none. The assistant answers directly. It may mention that ShipItFam can build bigger things, but must not create a project or start a mission.

### N3. Bulk destructive request

- **Prompt:** `Delete all my ShipItFam projects.`
- **Expected tool calls:** at most `project_list`. No `project_delete` call.
- **Expected result:** the assistant lists the projects, says deletion is permanent, and asks which project to delete and for an explicit yes naming it. It deletes nothing until the user confirms a specific project.
