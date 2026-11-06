---
name: use-openbot
description: How an agent running inside OpenBot should use the platform's own mechanisms, including parallel tasks for pushing several projects at once, reading your other threads, messaging teammates, long waits and wake-ups, interrupting, and installing skills. Use when new work arrives while you are busy, when you want to do several things at once, when you feel overloaded or tempted to queue work, when waiting on a long build, when delegating or replying to a teammate, or when installing or updating an OpenBot skill. Skip for small work that fits in one turn.
---

# Goal

One agent can push several projects at the same time. OpenBot gives you parallel tasks, so "I'm too busy" is not a reason to queue work, hand it back, or ask for another agent.

Most tools here are discovered first: call `openbot.tool_search` or `tool_describe`, then run them with `openbot.tool_call`. Trust the tool description for parameters. This page covers when to use which and what bites you.

## Where does this work go?

| Situation | Use |
|---|---|
| Fits in this turn, a few minutes | Do it in the main chat |
| Long and independent, or another project | `start_task` |
| Belongs to another role | `send_message` to a teammate |
| Lookup whose result this turn needs | Your provider's subagent |
| Build or deploy under about 15 minutes | Check it in this turn at growing intervals |
| Takes longer than this turn can wait | `await_event`, then end the turn |
| Timed or repeated | A routine (`create_routine`), not provider scheduling |

Order work only around things that are really shared: heavy jobs on one machine, one production window, the user's own time. Otherwise run it in parallel.

## Parallel tasks: `start_task`

Project A is running and project B needs work: open a task for B and keep the main chat free. One task per project is the easiest to track.

- The platform text says to start a task when the user asks for it. So get a standing instruction from your user, in your agent's instructions, that parallel tasks are the default for several projects.
- A turn started by a teammate's message can start a task too (tested).
- The `brief` is the task's whole context. It cannot see the chat. Include the goal, background, where the project files are, which risky actions are approved, and what done looks like.
- **The result does not come back to the main chat**, and a task cannot message its own agent (tested: `An agent cannot message itself`). So the brief must say how to finish: write results to a project file, and have the task `send_message` whoever needs them, with that agent's id and `replyToMessageId`.
- By default a task must not run git commands that change the branch, index, or history (commit, switch, reset, stash, rebase). If it should commit, say so in the brief.
- Save the returned `taskId` somewhere durable. It is how you check progress later.
- Tasks share the workspace with no file locking. Keep them off the same files. For code, give each task its own git worktree. Two jobs on one file belong in one task, one after the other.
- Tasks cannot start tasks (tested: `Only your own chat can start a task`), and channel turns cannot start them either.
- A task has its own browser tabs and no Computer Use. Keep desktop work in the main chat.
- Inside a task, `send_message`, `await_event`, `list_waits`, `history_read`, and `search_memories` all work (tested).
- Capacity on 0.62: an agent can start up to 8 tasks, and only running ones or ones with queued messages count. How many run at once is the "Tasks at once" setting under Advanced. The main chat does not count. Extra tasks wait.

## Reading your other threads

- `history_read` with `taskId` shows how a task is going. `includeWorkSteps: true` adds captured steps, with no raw reasoning.
- `history_search` with `taskId` and `query` searches that thread.
- Without `taskId` they read the current chat, which helps after context compaction.
- You only see your own threads, and only after the latest reset. Channels use `channel_history`. For a teammate's thread, ask the teammate.
- `list_agents` shows `status`, `queuedMessages`, and `lastActivityAt`. A teammate showing "working" can still take work: messages queue, and they can open a task.

## Teammates: `send_message`

- Messages are asynchronous and the recipient cannot see your chat. Put the goal, background, what you tried, approved risky actions, and what done looks like in one message.
- `expectsReply: true` (the default) is for requests and questions. The result comes back once in a later turn, so do your own part and end the turn. Do not wait with provider tools such as `wait_agent`.
- `expectsReply: false` is for information and for your reply to someone's request. Nothing comes back and the exchange ends.
- Answering a request: pass `replyToMessageId` and `expectsReply: false`.
- Replies are never forwarded to a third agent. Forward them yourself.
- Do not send a message only to thank someone or confirm receipt.

## Waits and wake-ups

- `await_event` opens a wait the user can see. Write a clear `note`, say what you wait for, and end the turn. The next run of one of your routines wakes it, or a local script event naming its `waitId`. Give `routineId` to accept only that routine.
- For a long command: create a paused routine that says what to do with the result, run the command detached, and call the routine's run endpoint after it with `;`, so a failure also wakes you.
- An agent can hold 20 waits. A script event without `waitId` wakes every wait that routine can wake, so pass the id.
- `list_waits` shows open waits (`agentId` for another agent). `close_wait` closes finished ones. A forgotten wait stays in front of the user.
- Each routine run is a new session that sees only the newest chat messages. Write instructions a stranger could follow: tools, accounts, who to report to, when to report nothing.

## Interrupting: `interrupt_agent`

It stops a teammate's turn only when that turn works on your message, and it cancels your queued requests. You cannot stop work that the user, a routine, a channel, or another agent started. To change the work, interrupt, then `send_message`. Do not interrupt only because something is slow.

When the user says stop: stop your own work, interrupt teammates on it, close waits, kill background jobs, then say what stopped and what may still run.

## Skills

To give every local agent a skill:

1. Put the skill folder in your workspace and call `create_skill` with `sourcePath`. It enters the local library and installs for you.
2. `list_local_skills` gives the `skillId` and `revision`.
3. `install_local_skill` with each `agentId`.
4. `read_agent` on each to confirm.

To update: `read_local_skill`, then `revise_skill` with `expectedRevision`. **Installed copies do not update on their own.** Run `install_local_skill` again for each agent with the new revision. Use `set_skill_enabled` to turn one off and `uninstall_skill` to remove it.

## Also worth knowing

- Call `read_agent` before changing an agent, and `update_profile` to change it. Only the user can grant full access, Computer Use, or auto-approve.
- Use `start_new_chat` when a chat is long and its past no longer matters. Save what you need first.

Tested on OpenBot 0.62. When this page and a tool description disagree, trust the tool description.
