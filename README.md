# elves

**A small loop for knowledge work: an agent that drafts before you ask, running on your laptop with scheduled jobs, a database you own, and a second-brain wiki you can read. Your only job is review.**

[![Ninety-second explainer](media/poster.png)](media/elves-explainer.mp4)

*Ninety-second explainer: [narrated](media/elves-explainer.mp4) · [silent, captions on screen](media/elves-explainer-silent.mp4) · [script](media/script.md)*

## Why share it now

[Muse](https://about.fb.com/news/2026/09/introducing-muse-personal-ai-agent/), [Grok Bot](https://x.ai/bot), and now [dots](https://openai.com/index/introducing-dots/) all sell the same shape of thing: an always-on agent that works while you're away, brings you finished work, and learns from your feedback. That shape is right.

This repo is not a competitor to any of them. It is a small loop I have been running on my own laptop since before they launched, and the reason to share it is that it is quick to set up and you can run it today with whatever coding agent you already use, including on a work machine where the consumer agents are not an option.

- **Scheduled jobs, not a cloud computer.** launchd or cron runs `/scan` hourly, `/ingest` nightly, and `/lint` weekly. It is always on while the laptop is awake, which turns out to be enough.
- **A database you own.** A sqlite file holds every task, its status, and every piece of feedback you give. Query it, back it up, delete it.
- **Full control.** It reads only the sources named in one `config.yaml`. Nothing is sent, shared, or changed outside the folder; every write tool a connector exposes is denied in the harness, not just discouraged in the prompt.
- **Any coding agent, including the one you already have at work.** You may not be able to use Muse, Grok Bot, or dots on a work machine, and you do not need to. The whole design is one markdown file, about 200 lines. Hand it to Claude Code, Codex, Cursor, or whatever your company already allows with MCP connectors, and it builds it.
- **A second brain you can read.** Everything the agent learns lands in a markdown wiki: a page per person, project, and output type, plus your style rules, an index, and a log. Every rule cites the feedback it came from. Open the folder in Obsidian and you can see all of it, and change any of it.

## The tale

The shoemaker goes to bed. In the morning, the shoes are finished. All he does is look them over.

That is the whole design. The agent fields requests, decides what deserves a draft, produces the draft without being told to start, and learns from your review so the next one is better. The executive's only job is `/review`.

## How it works

```mermaid
flowchart LR
  S["/scan<br/>hourly · reads only configured sources"] --> D["/draft<br/>immediately · nothing sent"]
  D --> R["/review<br/>after lunch · approve, edit, redraft, discard"]
  R --> I["/ingest<br/>feedback becomes rules in a wiki"]
  I -. every draft reads the wiki first .-> D
```

- **`/scan`** runs hourly from a scheduled job. It pulls new messages and upcoming events from the inbound sources in `config.yaml`, decides what needs action, and triages each thread: `draft`, `quick_reply`, `delegate`, `clarify`, or `ignore`. Ignored threads never become tasks. A meeting with other attendees is a request for prep; your own solo reminders become to-dos.
- **`/draft`** runs the moment a scan finds work. Each task gets a subagent that reads the wiki (style, the requester's page, the project's page, the playbook for that output type, and prior feedback) before writing anything, and writes its output to `tasks/<id>/`. No permission prompt. Nothing is sent.
- **`/review`** is the after-lunch view. Every item opens with the same line: *Action taken: read X, drafted Y at `tasks/<id>/…`, nothing sent.* You approve, edit, redraft, or discard. Each reaction is logged as a feedback row.
- **`/ingest`** turns feedback into rules. *"Too long, Hamin just wants the number"* becomes one line on `wiki/people/hamin.md`: *Prefers the headline number first; skip methodology unless asked.* Every rule cites the raw transcript it came from, and a newer rule rewrites an older one instead of piling up beside it.
- **`/lint`** runs weekly and flags stale pages, contradictions, and playbooks whose drafts keep getting low ratings.
- **`/todo`** is your own list: approved items you said you'd handle, drafts waiting for review, and what the agent is working on right now.

State lives in three places you can open directly: `tasks.db` (sqlite: tasks, feedback, runs, and the scan's dedupe memory), `tasks/<id>/` (the drafts), and `wiki/` (the rules). There is nothing else.

## The wiki is a second brain

The products above all say the agent "learns your preferences." Here you can read what it learned. The wiki is plain markdown with `[[wiki-links]]`, so it opens as an Obsidian vault, and it is organized the way a second brain is: by the people you work with, the projects you are on, and the kinds of things you produce.

```
wiki/
  index.md              every page, one line each
  log.md                one line per scan, draft, ingest, and lint
  style.md              your voice: tone, length, sign-off
  people/<name>.md      what each requester wants, distilled from your feedback
  projects/<name>.md    context, history, constraints, and which sources apply
  playbooks/<type>.md   how to produce a reply, a status update, a meeting prep
  lint/<date>.md        weekly findings
  raw/                  feedback transcripts, append-only, never edited
```

Three rules keep it honest. Feedback is distilled into a rule, never pasted in as a quote. Every rule cites the raw transcript it came from, so you can trace it. A newer rule rewrites an older one instead of accumulating beside it, so a page reads as what is true now. `/lint` flags the pages that have gone stale, contradict each other, or keep producing drafts you rate low. The result is a knowledge base that grows one review at a time and that you can open, search, and correct by hand whenever you like.

## Run it

The design is one markdown file, about 200 lines: [`automate-agent-task-execution.md`](automate-agent-task-execution.md). It covers scope, the setup interview, the database schema, each command, and the things that bit on the first day of running it unattended.

1. Clone this repo, or just copy the file into an empty folder.
2. Open the folder in a coding agent that can reach your tools (Claude Code, Codex, Cursor, or whatever you use with MCP connectors).
3. Tell it: *"Read `automate-agent-task-execution.md` and build it. Start with `/setup`."*
4. Answer the setup interview: which inboxes and calendars to watch, which folders it may read while drafting, who your regulars are.
5. Schedule `/scan` hourly (launchd, cron, or a systemd timer) with an explicit tool allowlist and a denylist for every write tool.
6. After lunch, run `/review`.

The seed is the product. The code your agent writes from it is yours to keep iterating on, and the seed is meant to absorb any lesson that would help someone else running the same loop.

## What good looks like after a month

- `/review` takes fifteen minutes and most items are approved unchanged. `/todo` takes fifteen seconds.
- The queue is only ever things that matter. Nobody scrolls past receipts to find the work.
- Meetings arrive with a one-page brief the day before, and your own reminders show up in `/todo` without anyone typing them in.
- The wiki has a page for every regular requester and active project, and the rules on those pages are things you would recognize as true.
- Average feedback rating is rising. If it isn't, `/lint` says why.

## What's in this repo

| Path | What it is |
|---|---|
| [`automate-agent-task-execution.md`](automate-agent-task-execution.md) | The seed. The entire design, meant to be handed to a coding agent. |
| [`media/`](media/) | The explainer video (narrated and silent), its captions, script, and poster frame. |

The running instance lives in a separate private folder, because `tasks/`, `tasks.db`, and `wiki/raw/` hold real messages, names, and verbatim feedback. Keep yours private too.

## Credits

The `/ingest` step is modeled on Andrej Karpathy's LLM wiki idea: the wiki is compiled knowledge, the agent is the compiler, and raw inputs are never edited. Built with Claude Code. A personal project by [Matthew Corritore](https://github.com/myaa2913).

## License

[MIT](LICENSE)
