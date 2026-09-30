# Automate Agent Task Execution

Imagine an executive returning from lunch to review a stack of drafted work for requests they didn't know had come in.

That's the goal. Professionals want time for proactive, high-value work, but collaboration tools like Slack and Gmail consume attention through constant monitoring, triage, and response. Agentic AI hasn't fixed this — many knowledge workers are now busier than ever triaging requests, delegating to agents, and babysitting them.

The fix is an agent that processes incoming work and drafts outputs *without being told to start*. It fields requests, decides what deserves a draft, produces the draft, and learns from the executive's feedback so the next draft is better. The executive's only job is to review.

This file is a parsimonious core loop. Give it to your coding agent and iterate.

## Scope and assumptions

- **Everything is local.** The database, the drafts, and the knowledge wiki live in `~/Documents/agents_productivity_system/`. Scheduled runs happen when the laptop is awake; that's acceptable for an hourly cadence. Do not build a cloud component.
- **Urgent items are handled elsewhere.** The executive's own notification settings in their communication tools already surface high-priority messages directly. This system handles everything that can wait for a batched review. Do not build an alerting or escalation path.
- **Nothing ships without approval.** The agent drafts; the executive approves. No message is sent, no file is shared, no external state is changed outside `/review`.
- **Drafting needs no approval.** Once `/scan` decides something deserves a draft, `/draft` runs immediately. The executive's gate is `/review`, not "shall I draft this?" — asking reintroduces the babysitting the system exists to remove.
- **Nothing is surfaced without drafted work.** Every item the executive sees comes with a file: a full draft, or at minimum a recommendation with next steps. The executive's first question is always "what did you do?", so every draft and recommendation opens with an *Action taken* line.
- **Sources and context are configured, not assumed.** Do not hardcode Slack, Gmail, or any other tool. `/setup` asks the executive what to monitor and what the agent may read, and writes the answers to `config.yaml`. Every other command reads its sources and permissions from there.

## `/setup` (run once, re-run to change)

An interview, not a form. The agent asks; the executive answers in plain language; the agent writes `config.yaml` and confirms it back.

1. **Inbound sources — what should I watch for requests?** List the connectors available in the current environment (MCP servers, CLIs, APIs) and ask which to monitor. For each one chosen, ask the scoping questions that connector needs: which channels or labels, which senders to include or exclude, whether DMs count, how far back to look on the first run. Anything not chosen is never read by `/scan`. A calendar counts as an inbound source: an upcoming meeting is a request for preparation, and the executive's own solo reminders are to-dos. For a calendar, ask which calendars, how far ahead to look, and which recurring titles to skip.
2. **Context sources — what may I read when drafting?** Same list, different question. These are the places a subagent may look beyond the wiki: document stores, calendars, ticket systems, code repos, the executive's own message history. For each, ask for the scope (a Drive folder, a repo, a date range) and whether it applies to every project or only some. Default for anything not named is *no access*. Include documents the executive has produced with AI assistants (chat-hosted docs, artifacts, notes); they are often the freshest statement of intent on a project. Every context source is read-only, so note the write tools each one exposes and deny them in the harness (see *Running it unattended*).
3. **Output types** — ask what kinds of work the executive expects to be drafted (replies, status updates, analyses, decks, code). Seed one `wiki/playbooks/<type>.md` stub per answer.
4. **Regulars** — ask for the handful of people and projects that generate most requests. Seed `wiki/people/` and `wiki/projects/` stubs so the first drafts aren't cold.
5. **Cadence** — how often `/scan` runs, and whether `/lint` runs weekly.

Write everything to `config.yaml`, show it to the executive, and ask them to confirm before anything runs. Re-running `/setup` edits the existing config rather than starting over.

```yaml
# config.yaml (shape, not a template — fields depend on the connectors chosen)
inbound:
  - connector: <name>
    scope: { ... }           # mail: labels, senders, lookback · calendar: calendars, lookahead, skip_titles
context:
  - connector: <name>
    scope: { ... }
    projects: [all] | [<project>, ...]
output_types: [...]
scan_interval_minutes: 60
ingest_nightly: true
lint_weekly: true
```

## Directory layout

```
agents_productivity_system/
  config.yaml               # written by /setup; the only place sources and permissions are defined
  CLAUDE.md / AGENTS.md     # hard rules and this executive's conventions, for whichever coding agent runs the commands
  schema.sql                # tasks.db DDL, idempotent
  tasks.db                  # sqlite, source of truth for task state (gitignored)
  tasks/<task_id>/          # one folder per task; drafted outputs live here (gitignored)
  scripts/                  # small deterministic helpers the commands call (e.g. the upsert)
  .claude/skills/<cmd>/     # one SKILL.md per command: setup scan draft review ingest lint todo (or .agents/skills/ — whatever your coding agent reads)
  launchd/                  # scheduler job definitions and their stdout/stderr logs (cron or systemd elsewhere)
  wiki/                     # LLM-maintained knowledge base (see below)
    raw/                    # immutable source captures (feedback transcripts, reference docs) (gitignored)
    index.md                # catalog of every wiki page with one-line summaries
    log.md                  # append-only chronological record of every ingest, draft, and lint
    schema.md               # conventions for the wiki itself
    style.md                # the executive's voice and formatting rules
    people/<name>.md        # one page per requester or frequent collaborator
    projects/<project>.md   # one page per project
    playbooks/<type>.md     # how to produce a given output type (status update, analysis memo, reply, recommendation, meeting prep)
    lint/<date>.md          # weekly findings from /lint
```

If the directory is a git repo, keep the personal layers out of it: `tasks.db`, `tasks/`, and `wiki/raw/` hold message content, names, and verbatim feedback. The wiki's distilled pages and the skills are what's worth versioning. If more than one coding agent will run the commands, keep one instructions file per agent with the same content; this seed stays agent-agnostic.

## Database

`tasks` table:

| field | notes |
|---|---|
| `task_id` | primary key |
| `source` | connector name from `config.yaml`, or `manual` |
| `source_ref` | thread/message ID or URL; `(source, source_ref)` is the dedupe key |
| `requester` | who asked |
| `title`, `description` | |
| `project` | links to `wiki/projects/<project>.md` |
| `output_type` | links to `wiki/playbooks/<type>.md` |
| `importance` | `low` / `medium` / `high` |
| `due_date` | nullable |
| `triage` | `draft` / `quick_reply` / `delegate` / `clarify` — never `ignore`; ignored threads don't become tasks |
| `status` | `pending` → `drafting` → `drafted` → `approved` / `rejected` → `shipped` |
| `output_path` | `tasks/<task_id>/` |
| `queued_text` | for `quick_reply` / `delegate` / `clarify`: the suggested line(s) |
| `date_added`, `date_modified` | |

`tasks` holds only things that need the executive's attention. `SELECT * FROM tasks` is the queue, nothing else. `shipped` means "the executive did it" — set by `/todo done`, or by `/scan` when the inbound source shows it happened (a payment confirmation, a booking email).

`seen` table: `source`, `source_ref`, `last_message_at`, `triage`, `seen_at`. For a message thread `last_message_at` is the newest message; for a calendar event it is the event's `updated` timestamp, so a changed invite is re-judged and an unchanged one is skipped. The scan's short-term dedupe memory — every thread a scan looks at gets a row, ignored or not, so it isn't re-judged next hour. A thread is re-evaluated only if it reappears with a newer message. Pruned to 24 hours at the end of every scan; it stays a few dozen rows. (An early version kept ignored threads in `tasks` instead. Within a day the queue was 97% receipts and newsletters. Don't.)

`feedback` table: `feedback_id`, `task_id`, `timestamp`, `rating` (1–5), `action` (`approve` / `edit_approve` / `redraft` / `discard`), `text`, `ingested_at` (null until `/ingest` has distilled the row). One task can have many feedback rows.

`runs` table: `run_id`, `command`, `started_at`, `finished_at`, `notes`. One row per command invocation. `/scan` uses the last *finished* scan's `started_at` as its "since" watermark, so an hourly cadence reads the last hour and a laptop that slept for five hours catches up automatically.

## The loop

### 1. `/scan`

- Pull new messages since the last finished scan (the `runs` watermark) from every inbound source in `config.yaml`, respecting each source's scope. Read nothing that isn't configured. Use the source's most precise time filter (Gmail accepts epoch seconds in `after:`; a date-granular filter re-fetches the whole day every hour). Always exclude the executive's own address — their other agents and automations email them too, and self-sent mail is never a request.
- Skip anything already in `seen` unless it has a newer message. Only new or updated threads get read in full and classified.
- For each thread, decide whether it needs action. If so, classify it:
  - `draft` — produce a substantive work output. Goes to `/draft`.
  - `quick_reply` — one or two lines. Write the suggested reply into the task's recommendation; don't spin up a subagent.
  - `delegate` — someone else should own it. Write a forwarding note naming who.
  - `clarify` — too ambiguous to draft. Write the clarifying question, plus the agent's best guess.
  - `ignore` — needs no action. Recorded in `seen` only; never becomes a task.
- **Calendars are inbound too.** Read events on the configured calendars from now to the end of the lookahead window (48 hours works). `source_ref` is the event id; `last_message_at` is its `updated` timestamp. An event with at least one other attendee is a request for prep: `draft`, output type `meeting-prep`, due on the event's start date, importance high inside 24 hours. A solo event that reads like a task ("call…", "renew…", "pay…") is the executive's own note to themselves: it becomes a task born `approved` with a recommendation file, so it shows in `/todo` and skips review. Solo events that aren't task-like (focus blocks, lunch, travel) are `ignore`. A cancelled event whose task already exists is set `rejected`. Recurring casual events with family or friends get no prep even though they have attendees; their exact titles go in a `skip_titles` list in the config. Prep is for events where the executive has to say or decide something.
- Every non-ignore, non-draft task gets `tasks/<task_id>/recommendation.md` in the same run: *Action taken · What it is · Recommendation · Next steps · What I'd do next time*. That last section is the rule the item would become if approved, so `/ingest` can lift it verbatim.
- Upsert into `tasks` on `(source, source_ref)`. Existing rows update in place; never touch `status` on conflict. Record every thread in `seen`, then prune `seen`.
- If any task is now `draft` + `pending`, run `/draft` immediately — don't wait for the executive. Runs hourly via a scheduled job, or on demand. Every run must be idempotent — running it twice does nothing the second time.
- The executive can also add tasks manually through the agent ("add a todo: call the dentist"). Those go in with `source = manual`, already `approved`, and a minimal recommendation file, so `/todo` shows them.

### 2. `/draft`

- For each task with `triage = draft` and `status = pending`: atomically flip `status` to `drafting`, then spawn a subagent. Run subagents in parallel where tasks are independent. The atomic claim prevents two runs from drafting the same task.
- Each subagent, before writing anything, reads in this order:
  1. `wiki/index.md` — to find relevant pages
  2. `wiki/style.md`
  3. `wiki/people/<requester>.md` if it exists
  4. `wiki/projects/<project>.md` if it exists
  5. `wiki/playbooks/<output_type>.md` if it exists
  6. All `feedback` rows for prior tasks with the same requester, project, or output type
- Context beyond the wiki comes only from the `context` sources in `config.yaml`, filtered to those whose `projects` list includes this task's project. A project page in the wiki may narrow that scope further (a specific folder, a specific channel) but never widen it. If nothing in the config applies, the subagent is wiki-only and says so in its README.
- Don't invent context. If the wiki and the allowed sources say nothing about a person, write "no prior contact on record" rather than mining years of mail to build a profile for a social event. For a recurring meeting, start from the previous occurrence's draft (its `tasks/` folder, if present) and lead with what changed.
- Write outputs to `tasks/<task_id>/`, organized as the playbook specifies. Include a short `README.md` in the folder: what was requested, what was drafted, what the subagent was unsure about.
- On completion: `status = drafted`, `output_path` set, one line appended to `wiki/log.md`.

### 3. `/review`

The after-lunch view. Invoked by the executive.

- Every item leads with one line: *Action taken: read X, drafted Y at `tasks/<id>/…`, nothing sent.* The executive's first question is always "what did you do?" — answer it before showing anything else. Never surface an item without a drafted file.
- Show everything `drafted` since the last review, ordered by `importance` then `due_date`. Include `quick_reply`, `delegate`, and `clarify` items as a short list at the top — each with its recommendation sentence; they take seconds.
- For each item the executive chooses one of: **approve / edit then approve / redraft / discard**. Approve marks `approved`; the executive ships it themselves (or a later version of this system ships it on their behalf).
- As the executive reacts, log every reaction as a `feedback` row. Prompt for a one-line reason on anything rated 3 or below, or on any discard.
- After the session, run `/ingest` on the feedback collected.

### 4. `/ingest`

The step that makes the system improve. Modeled on Karpathy's LLM wiki: the wiki is compiled knowledge, the agent is the compiler, and the raw inputs are never edited.

- Runs at the end of every `/review`, and again nightly as a sweep. Each run processes only feedback rows not yet stamped `ingested_at`, so feedback given outside a review (a comment in chat, an interrupted session) still reaches the wiki within a day, and a run with nothing new is a no-op.
- Save the batch's feedback transcript to `wiki/raw/feedback/<date>.md`. Raw files are append-only and never modified.
- Read `wiki/index.md` and `wiki/schema.md`.
- For each piece of feedback, decide which page(s) it changes:
  - A comment about voice, tone, length, or formatting → `style.md`
  - A comment about what a specific person wants → `people/<name>.md`
  - A comment about a project's context, history, or constraints → `projects/<project>.md`
  - A comment about how a type of output should be structured → `playbooks/<type>.md`
  - A comment about what should be surfaced at all ("never prep games night", "that sender is always noise") → the source's scope in `config.yaml`: a skip list, an excluded sender. Scope edits from feedback only narrow; they never grant new access. Log them like any other rule.
- **Distill, don't append.** Feedback becomes a rule, not a quote. "This is too long, Sam just wants the number" becomes, on `people/sam.md`: *Prefers the headline number first; skip methodology unless asked.* Each rule cites the raw file it came from (`[feedback/2026-09-25.md]`) so it can be traced and, if later contradicted, revised rather than duplicated.
- If a new rule contradicts an existing one, the newer wins and the old one is rewritten, not left alongside.
- Create new pages when a requester, project, or output type appears for the first time. Every page gets a line in `index.md`.
- Link pages with `[[wiki-links]]` where they relate (a person to their projects, a project to its playbooks).
- Append one line to `wiki/log.md`.

### 5. `/lint` (weekly, scheduled)

Wiki maintenance so it doesn't rot.

- Flag pages not updated in 60+ days whose subject has had recent tasks.
- Flag contradictions between pages (a style rule on `style.md` vs. a person-specific override that isn't marked as one).
- Flag orphan pages with no inbound links and no matching tasks.
- Flag rules in the wiki that recent feedback ratings suggest aren't working (a playbook whose tasks average ≤ 3).
- Write findings to `wiki/lint/<date>.md` for the executive to skim during the next `/review`. Fix only what's mechanical (broken links, index entries); leave judgment calls for the executive.

### 6. `/todo`

The executive's own outstanding list — distinct from `/review`, which is about the agent's work. Invoked whenever they want to know "what do *I* need to do?" One query, one screen, no preamble:

1. **Do these** — `status = approved`, not yet `shipped`. Things the executive said they'd handle themselves, plus items they created themselves (manual to-dos, solo calendar reminders), which land here already `approved`. Each line: short id · importance · due date (flag anything within two days or past) · title · the recommendation sentence · a `/todo done <id>` hint.
2. **Waiting for your review** — `drafted` items and undecided `quick_reply` / `delegate` / `clarify` items. Ends with "run `/review`".
3. **Agent is working on** — `drafting`. No action needed.

Empty sections are omitted; if everything is empty it says "Nothing outstanding." and stops.

`/todo done <ref>` sets `shipped` and logs a `note` line. A ref is a short id or a word or two from the title (`/todo done rent`); if a title fragment matches more than one approved item, list the candidates and ask rather than guess. It refuses anything that isn't `approved`, so it can't be used to skip the review gate. `/scan` may also set `shipped` on its own when the inbound source proves an approved item happened.

## Running it unattended

The loop only pays off if `/scan` runs without anyone at the keyboard. Things that bit on the first day, so you don't have to rediscover them:

- **Scheduling.** A local scheduler (launchd on macOS, cron/systemd timers elsewhere) running `claude -p "/scan"` from the project directory hourly, `/ingest` nightly, and `/lint` weekly. Log stdout/stderr to files you can read. Schedule by wall clock (on launchd, `StartCalendarInterval` at minute 5 of every hour) rather than by elapsed interval; the interval form never fired unattended in testing.
- **Permissions.** A headless run can't answer a permission prompt — it just exits. Pass an explicit tool allowlist: the sqlite command, the upsert script, file read/write, and the *read-only* tools of each configured connector. Then pass a denylist naming every write tool each connector exposes: send / reply / forward / draft / label / trash for mail, create / update / respond / delete for calendars. "Nothing ships" should be enforced by the harness, not only by instruction. If a tool is denied mid-run anyway, the command logs the denial in the run's notes, finishes what it can, and exits cleanly; a headless run that blocks on a question looks exactly like a dead scheduler.
- **Tool names differ by host.** A connector that appears as `mcp__<uuid>__<tool>` in a desktop session can appear as `mcp__<ConnectorName>__<tool>` headlessly. Store connector *names* in `config.yaml`, not literal prefixes, and let each command resolve the tool at runtime.
- **Auth expires quietly.** The CLI can report "logged in" while its token is rejected. Watch the log for 401s after any gap; re-login is a one-line command, but it needs a human with a browser.
- **Verify the first scheduled fire, not just a manual kick.** A manually triggered run proves the job is well-formed; only a run you didn't start proves the scheduler is doing its job.
- **Quiet hours still get a log line.** Every scan appends one line to `wiki/log.md` even when nothing was actionable (fetched n, ignored n, 0 new tasks). Without it you can't tell a quiet inbox from a scheduler that stopped firing.
- **Idempotency is the safety net.** With a `runs` watermark and a `seen` table, a job that fires twice, or a manual run overlapping a scheduled one, does no harm.

## What "good" looks like after a month

- `/review` takes fifteen minutes and most items are approved unchanged. `/todo` takes fifteen seconds.
- The queue is only ever things that matter. Nobody has to scroll past receipts to find the work.
- Meetings arrive with a one-page brief the day before, and the executive's own reminders show up in `/todo` without anyone typing them in.
- The wiki has a page for every regular requester and active project, and the rules on those pages are things the executive would recognize as true.
- Average feedback rating is rising. If it isn't, `/lint` says why.
