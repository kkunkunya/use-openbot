# use-openbot

A skill that teaches agents running in [OpenBot](https://openbot.run/) to use OpenBot's own mechanisms, so one agent pushes several projects in parallel instead of queuing them.

Agents get tool descriptions, which say what each tool does. They do not say when to reach for it or what goes wrong. This skill fills that gap with a decision table and the pitfalls, several of them tested on OpenBot 0.62:

- A task's result never returns to the main chat, and a task cannot message its own agent.
- Tasks share the workspace with no file locks.
- Tasks are told to leave git history alone unless the brief says otherwise.
- Installed skills do not update when the library does.

It covers `start_task`, `history_read`, `send_message` and `expectsReply`, `await_event`, `interrupt_agent`, and local skill installs.

## Install

Ask one of your OpenBot agents:

> Copy `skills/use-openbot` from this repo into your workspace, register it with `openbot.create_skill`, install it for every agent with `openbot.install_local_skill`, and confirm with `openbot.read_agent`.

Then add one line to your agents' instructions, because OpenBot's own guidance says to start tasks only when the user asks:

> When several projects need work, use parallel tasks by default.

## Notes

This is a community skill, not made by or affiliated with the OpenBot team. Official sources: [openbot.run](https://openbot.run/) and [nightly-labs/openbot](https://github.com/nightly-labs/openbot).

MIT licensed. See `LICENSE`.
