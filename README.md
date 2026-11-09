# AI Inbx skills

Agent skills for [AI Inbx](https://aiinbx.com), the email API for software that holds conversations. They take a coding agent from "add email to this app" to the app sending an email, receiving the reply and answering it in the same thread.

## Install

```sh
npx skills add aiinbx/skills
```

Or hand your agent one line:

```text
Set up AI Inbx in this project. Read https://aiinbx.com/SKILL.md and follow it.
```

## Skills

| Skill                                                  | Job                                                                                           |
| ------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| [`aiinbx-setup`](skills/aiinbx-setup/SKILL.md)         | First-time setup: CLI, sign-in, key, and one verified round trip in your app                  |
| [`aiinbx-integrate`](skills/aiinbx-integrate/SKILL.md) | The app's server code: sending, a verified webhook, threads, replies, drafts, delivery events |
| [`aiinbx-concepts`](skills/aiinbx-concepts/SKILL.md)   | The objects: domains, mailboxes, emails, threads, events, spaces                              |
| [`aiinbx-cli`](skills/aiinbx-cli/SKILL.md)             | The `aiinbx` command line: send, `listen`, `wait`, `doctor`, every API resource               |

`aiinbx-setup` runs the flow and hands over to the others, so installing all four is the usual choice.

## Where this comes from

Each `SKILL.md` is written by hand. The files under `references/` are pages of [docs.aiinbx.com](https://docs.aiinbx.com), copied in when the docs change, so a reference never says something the docs don't.

This repository is generated. Issues are welcome; edits get ported into the source.

## License

MIT
