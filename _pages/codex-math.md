---
permalink: /codex-math/
title: "Using Codex to attack math problems"
excerpt: "For mathematicians and TCS people who have never used an AI coding agent."
author_profile: false
sitemap: true
---

_For mathematicians and TCS people who have never used an AI coding agent._

_Author: Elfarouk Harb._

----------

## 0. A forewarning

With the current speculations around OpenAI using user's math data as training data, I'd highly recommend to login to your ChatGPT, click on your name in the bottom left corner, click settings, go to data controls, and set "Improve the model for everyone" to off so it does not train on your data/ideas for your problem.

## 1. What this is

Codex is an OpenAI harness/model that runs in your terminal. It reads your files, writes scripts, runs them, reads the output, does long stretches of mathematical reasoning, and iterates. Run that loop for a few weeks on a problem you care about and it can make real progress.

The mental image I find useful is a very fast, tireless, well-read postdoc who implements ideas immediately, doesn't mind tedious computation, and is overconfident enough to hand you a beautiful proof of a false statement without blinking. You'd still hire them. You'd also check their work.

### Why the terminal and not the ChatGPT box

You can paste a problem into ChatGPT and have a useful conversation. The reason to use the terminal tool instead is that it runs code, keeps files, and can launch several agents to explore different ideas at once. In the chat box you get one agent, no persistent memory, and a context window that eventually runs out and forces you to start a new chat session. In the terminal the agent keeps a research directory: the scripts it used to stress-test conjectures, a running account of what has been tried, a folder of relevant papers, and its current state of play. A proof attempt that spans several weeks doesn't fit in one conversation. It has to live in files on your disk that the agent rereads each session.

Your ChatGPT chat allowance is separate from your Codex allowance, so when you run out of one you can fall back on the other. In my experience Codex is much stronger for this kind of open-ended hard research work and exploring many ideas in parallel, but the chat box is the right place for "is this inequality true?" or "Prove this very isolated and straightforward technical Lemma" and similar background questions, and those cost you nothing from the Codex budget.

----------

## 2. Setup

You need a paid ChatGPT plan (Pro, from $100/month, to $200/month, if this becomes a daily tool), a terminal on macOS or Linux (on Windows, install inside WSL), and Node 18+ if you go the npm route.

```bash
npm install -g @openai/codex   # or: brew install codex
codex login
```

Then make a directory for the problem and start there. The directory you launch from is the agent's workspace.

```bash
mkdir ~/research/my-problem
cd ~/research/my-problem
git init        # do this, trust me
codex
```

**Understand the sandbox first.** By default Codex can read, edit, and run commands inside your workspace without asking, and stops to ask before writing outside it or touching the network. Your OS is the entity that enforces that boundary, not the model's good intentions.

Inside Codex, `/status` shows the active model, the approval policy, and which directories are writable. `/permissions` changes the preset mid-session.

----------

## 3. Four things worth understanding

**Tokens.** Roughly ¾ of a word each. Everything is measured in them. This includes your prompt, the files it reads, the output of every command, its internal reasoning, its reply, etc. A 40-page PDF is about 30,000 tokens for reference.

**A turn.** You type something to codex, it thinks, then reads files and runs commands and edits things for anywhere up to several minutes, then reports back. That whole stretch is one "turn". A session is made of multiple turns.

**The context window.** Its working memory, around a million tokens on current models. Everything in the session lives there. The session is essentially the concatenation of all turns so far in this session. Two things follow. 1) Quality drops as the session fills up, because a window stuffed with junk has worse reasoning ability  than a fresh one, unless of course the session is full of useful information. 2) When the session does fill up to the 1M context window, `/compact` (or automatic compaction) replaces the old turns with a summary to free up space in the context window. It essentially replaces every turn with a "summary" of it, and overwrites the old long context window with the new concatenation of summaries as your current session.

**It forgets everything between sessions.** If you close the terminal and the conversation is gone, a new session starts from scratch. `/resume` reloads a transcript and there's an optional memory feature, but the real answer for persistency is that the files on disk are the **memory**. You keep a few plain-text files describing the state of the research, and every "turn" in the session starts by reading them. Section 5 explains my current setup.

----------

## 4. Your first session

Pick a statement you already know is true, ideally one with a finite check, and ask it to verify the thing exhaustively.

Something like:

```
Conjecture (the 1/3–2/3 conjecture; Kislitsyn, Fredman, Linial). Let P be a finite
poset that is not a total order. For incomparable x, y let p(x<y) be the fraction of
linear extensions of P in which x precedes y, and set

    delta(P) = max over incomparable pairs {x,y} of min( p(x<y), p(y<x) ).

Then delta(P) >= 1/3 for every such P.

Enumerate all non-isomorphic posets on n elements, for n = 3 upward until it stops
being feasible, and compute delta(P) exactly for each. Count linear extensions exactly
and keep p(x<y) as a rational.

Report a table: n, the number of posets you enumerated, the minimum of delta over them,
and the poset attaining that minimum, given as a covering relation.

If the bound survives everything you can enumerate, attempt a proof from first
principles, and be explicit about what you are actually proving. The general conjecture
is open and the best published general bound is around 0.2764,
Then stop and show me the table and whatever proof you produced.
```

----------

## 5. Setting up a real project

```
my-problem/
├── AGENTS.md          standing instructions, read automatically every session
├── recap.md           living summary of the project
├── research_log.md    append-only experiment ledger
├── plan.md            proven / verified but unproven / next
├── papers/            PDFs, plus summaries.md written by the agent
├── experiments/       one script per hypothesis
└── writeup/           LaTeX
```

Codex reads `AGENTS.md` from your project root automatically at the start of every session, so anything in it becomes standing instruction. Run `/init` to get a scaffold, then replace it with your own rules. Mine are roughly:

````markdown
# Research protocol

An attempt at one open problem. Read recap.md before doing anything.

## Files to maintain
- research_log.md: append only, never edit past entries. Per experiment: number,
  date, script, the hypothesis in one sentence, a table of actual numbers, a verdict
  (confirmed / refuted / inconclusive), and what it implies for the strategy.
- recap.md: problem statement, proven lemmas with sketches, current strategy and
  what blocks it, failed approaches and why each failed, index of scripts.
- plan.md: what is proven, what is verified computationally but unproven, next step.

## Rules
- Compute before proving. Never attempt to prove something unverified on small cases.
- Numbers, not adjectives. "The bound seems to hold" is useless.
- Report the size of every search space you cover, as a number.
- Exact arithmetic wherever the conclusion depends on it.
- One hypothesis per experiment, one script per experiment, descriptive filenames.
- Record precise failures: which step breaks, for which parameters, with a witness.
- Nothing is "proven" until it has survived two rounds of adversarial review.
- If you can't verify a claim, say so instead of writing "it can be shown that".

## Ask me
I have better intuition about which directions are promising than you do. When
you have several plausible options and no strong reason to prefer one, ask instead
of guessing, and tell me if you think a direction I suggested is a dead end.
````

Keep it under a page. It gets prepended to every message you send.

Then pin the objective with `/goal`, which keeps a target in view across a long run of several turns:

```
/goal Prove [conjecture], or find a counterexample. The [special case] is known;
the general case is open.

```

`/goal` on its own shows the current goal, and there are `/goal edit`, `/goal pause`, `/goal resume`, and `/goal clear` variants. Without the goal, agents start drifting: you ask about a lemma, it notices your plotting code is ugly, and forty minutes later it has refactored your plotting code and hasn't done any work on tackling your lemma.

As for the three files, the point of `research_log.md` is that failures become reusable. After several compactions of your session, the agent forgets old approaches, and starts proposing the same arguments for the millionth time, the log is what lets it say we tried that in experiment 7 and it breaks for odd n. `recap.md` is the briefing you'd write for a collaborator taking over tomorrow, and it's what makes a fresh session/compacted new session useful. `plan.md` exists to keep statements that are _proven_ and _verified up to n=12_ apart, because that distinction blurs fast in long conversation. Insist the log is append-only; agents like to tidy history retroactively.

----------

## 6. The opening prompt

With `AGENTS.md` and `/goal` in place, the first message only needs the problem-specific part.

````markdown
# The problem
[State the conjecture precisely and define every term. Don't assume it knows your
notation or your field's conventions.]

[Any reductions or reformulations you have.]

[The techniques you already tried and precisely why each failed. Not "induction
doesn't work" but "induction on n fails because the step needs the extremal object
for n-1 to be a restriction of the one for n, and it isn't for n=7".]

# Prior art
The relevant papers are in ./papers/. For each: the exact statement proved, the
technique, and whether that technique could transfer to our formulation, with a
specific reason if not. Cite file and page. Write it to papers/summaries.md.

# This session
Ground truth only. Enumerate everything feasible, solve each instance exactly, and
show me the extremal objects rather than just their values. For every quantity you
compute, check whether it's always an integer, always positive, monotone in some
parameter, matches a closed form, or appears in OEIS. Log it all with real numbers.

# Attitude
Let the experiments surprise you. The useful discovery is usually something nobody
was looking for: unexpected linearity, unexpected zeros, a constant that keeps
showing up.

If you think my reduction is wrong or my statement is ambiguous, say so now.
````

----------

## 7. Model and thinking effort

To choose a model, you can run  `/model` which lets you pick a model, then a reasoning level.

The reasoning ladder runs `low`, `medium` (default), `high`, `xhigh`, `max`, `ultra`. Use `high` or `xhigh` for tracing an argument or checking edge cases, `max` for the genuinely hard step when depth beats speed and cost. `ultra` is different in kind: it splits the task across **subagents** automatically, so it's for work that actually decomposes ("attack this by four distinct techniques"), not a generic "try harder". If `max` doesn't appear in your picker, turn it on in app settings.

For mathematics the effort dial matters more than the model dial. Going from `high` to `max` on the blocking lemma buys you more than switching families. I'd use GPT-6-astra (`-m gpt-6-astra`) for the hardest sustained multi-step work, GPT-5.6 Sol (`-m gpt-5.6-sol`) for open-ended proof work.

----------

## 8. What you're paying for

Charging is per token. Your plan includes an allowance measured in "local messages" per rolling five-hour window, plus weekly limits. On Plus that's roughly 10–100 messages per window on Sol and 250–2,000 on Luna; Pro multiplies those by 5 or 20. Beyond the allowance you pay in credits, and Plus and Pro can buy them without upgrading. A turn already in flight is allowed to finish.

Every turn re-sends the whole conversation. What stops that being quadratic cost is **prompt caching**: if the beginning of the request is byte-for-byte identical to a recent one, the model reuses its precomputed state and bills those tokens at 10% (for those who are more technical, if the KV cache of your conversation is in OpenAI's servers, you only get billed at 10% the cost). Codex is built to keep that stable prefix long, appending new content only at the end. But it's an exact prefix match, so anything that changes early content silently throws the discount away. A cache miss will not show an error, you just start paying ten times more. The usual culprits are editing `AGENTS.md` mid-session, attaching or restarting an MCP server mid-session, `/compact` (which writes a fresh summary and cold-starts the cache), spawning a subagent (each child starts cold), and long idle gaps, since caches go cold after a few minutes.

So: keep `AGENTS.md` short and stable, keep volatile material in your messages and in files read on demand, and prefer one long session to twenty scattered ones. `/status` shows your remaining context and limits; `/usage weekly` shows the account view.

Two settings worth knowing:

```toml
# ~/.codex/config.toml
[features.rollout_budget]
enabled = true
limit_tokens = 2_000_000
```

And on Plus or Pro with Astra there's an experimental context mode that keeps notes across context windows and can search earlier messages and tool results from the same task, instead of repeatedly compressing everything into one summary. For a multi-week search it's worth trying:

```toml
# ~/.codex/config.toml
[features.context_management]
experimental_mode = true
```

----------

## 9. Running several agents in parallel

Codex can spawn child agents that work on separate pieces of a task at the same time, each with its own context window, then collect their results. This is on by default now.

The obvious benefit is speed, four techniques attempted at once instead of in sequence. The less obvious one matters more: it keeps noise out of your main conversation context window. Your main session should hold the conjecture, the constraints, the decisions and the mathematics, not 900 lines of enumeration output from a python script. Subagents absorb the mess and hand back a summary, so the main thread stays sharp for longer.

It isn't free. Each child does its own model and tool work, so this uses more tokens than a single-agent run (the recent Navier-Stokes result by OpenAI used 10,000 agents in parallel for reference), and six agents doing thirty seconds of work each costs more than one agent doing three minutes. Delegate when the pieces are genuinely independent and substantial. Parallel _reading_ is safe; parallel _writing_ causes conflicts, so give each agent its own output directory.

```toml
# ~/.codex/config.toml
[agents]
max_concurrent_threads_per_session = 8
default_subagent_model = "gpt-5.6-luna"
default_subagent_reasoning_effort = "high"

```

`max_concurrent_threads_per_session` is the dial for how many run at once. Eight is plenty; past that you're mostly paying for agents to wait on each other.

To define your own agent, put one TOML file per agent in `.codex/agents/` for the project or `~/.codex/agents/` for yourself. `name`, `description`, and `developer_instructions` are required; `model`, `model_reasoning_effort` and `sandbox_mode` are the useful optional ones, and anything you omit is inherited. The two I use most:

```toml
# .codex/agents/prover.toml
name = "prover"
description = "Attempts one specific lemma by one specific technique."
model = "gpt-5.6-sol"
model_reasoning_effort = "max"
sandbox_mode = "read-only"
developer_instructions = """
You get exactly one lemma and one technique. Don't switch techniques and don't
attempt a different lemma.
Before claiming a proof, verify every inequality numerically on the smallest
non-trivial cases and say which steps you checked and how.
If the technique fails, stop early and report the obstruction precisely: which step
breaks, for which parameters, with a counterexample or numeric witness. A clean
failure report is a successful run.
At most 400 words plus the proof or the obstruction. No preamble.
"""

```

```toml
# .codex/agents/refuter.toml
name = "refuter"
description = "Hunts for counterexamples."
model = "gpt-5.6-luna"
model_reasoning_effort = "max"
sandbox_mode = "workspace-write"
developer_instructions = """
Your job is to break the claim, not support it.
Write a self-contained script under experiments/refuter/ and run it. Exact
arithmetic. Enumerate exhaustively for small parameters before sampling randomly.
Report the extremal instance found, its exact value, and the size of the space you
covered as a number, never as an adjective. If nothing breaks, report the extremal
value observed and the exact count of cases checked.
"""

```

Variations on the same pattern are worth having: a `computer` agent for enumeration and exact solving, a read-only `librarian` for the papers folder, and a `critic` holding the referee prompt. You can also add problem-specific agents that run certain scripts/verify things in your own problem with additional context (for example `integrality_gap` agent for checking the integrality gap of your LP solution)

Delegation happens when you ask for it, so be explicit about the split, whether to wait, and what to return:

```
Attack [Lemma] with parallel agents. Spawn one prover per technique and wait for
all of them:
  1. [technique]
  2. [technique]
  3. [technique]
In parallel, spawn one refuter to search small cases for counterexamples.
Each agent writes only to experiments/<agent-name>/.
When they're all done, give me one table: technique, verdict, precise obstruction,
smallest parameter where it breaks. No prose summary.

```

One agent per item, wait for all, summarise per item. That's the pattern that works.

While they run, `/agent` switches between threads and `/ps` shows background terminals. Set your `/permissions` preset _before_ delegating, because children inherit the parent turn's live sandbox and approval settings even when an agent file says otherwise.

----------

## 10. Commands you'll actually use

`/goal` sets the persistent objective, `/plan` gets a strategy before anything is touched, `/model` sets model and effort, `/status` shows model, sandbox, context and limits, `/usage` shows token usage.

`/diff` shows everything it changed, including untracked files, and you should use it constantly. `/review` reviews the working tree and behaves like a referee if you point it at a proof.

`/side` opens a throwaway side conversation off the current one, which is where "wait, is Cauchy–Schwarz even tight here" belongs. `/fork` clones the conversation, which is how you pursue two strategies from the same state without losing either. `Esc Esc` on an empty prompt lets you edit your last message and branch from there. `/rename` names the session so you can find it again, and `/resume` reopens one.

`/compact` frees context, `/mention` attaches a file (or type `@` to search), `/init` generates `AGENTS.md`, `/permissions` changes the sandbox preset, `/agent` and `/ps` handle subagents and background terminals.

Mid-turn: `Enter` injects an instruction now, `Tab` queues one for later, `Ctrl+O` copies the last output, and `!cmd` runs a shell command yourself without leaving the session.

----------

## 11. General Prompting Tips and Gotchas

**1. It will produce a beautiful proof of a false statement.** This is the central risk and better models don't remove it. Verify lemmas before believing them, make it say which steps it checked, and referee from a session that didn't build the proof.

**2. It quietly weakens the statement.** Re-read the final statement against what you actually asked. Sometimes it changes the problem mid-session :)

**3. Floating point will lie to you.** An exact 1/3 shows up as 0.33333333333333337 in floating point arithmetic. Demand exact rationals wherever the conclusion depends on it, and put that in `AGENTS.md` so you stop having to say it.

**4. It agrees with you, and that's the enemy.** Say "I think the exchange argument works" and it will try to make the exchange argument work. Don't ask "does this work?", ask for the strongest objection, or what would have to be true for it to fail. Asking "are you sure?" is worse than useless because usually it will flip its answer. Instead, demand a specific check instead.

**5. Citations and "standard" facts can be invented.** It will attribute a theorem to a paper that proves something else, or call a false lemma standard. Make it cite file and page from PDFs you actually gave it, and treat every uncited "it is well known that" as unproven until you verify it yourself.

**6. It reward-hacks your checks.** Asked to make a failing check pass, it may special-case the failing input, loosen a tolerance, or quietly narrow the range being tested. When something that was failing starts passing, ask what changed in the test before you ask what changed in the mathematics.

**7. Editing `AGENTS.md` mid-session multiplies your costs.** It's in the cached prefix. Edit it between sessions.

**8. Use git and commit often.** The sandbox protects `.git` but not your uncommitted files, and an agent asked to tidy up the experiments directory will do exactly that.

**9. Long sessions get worse before they get full.** If answers are going vague and it's forgetting constraints you set an hour ago, that's the signal: update your files, start fresh (/compact), and delegate the noisy work next time.

**10. It never tells you it's stuck.** It will generate plausible next steps indefinitely. The rule that after about five failed approaches you stop trying techniques and go compute something you haven't computed has to come from you.

Two smaller things. It will cheerfully reinvent a result from 1974, so a literature pass and a quick "is this known?" in the chat box are cheap insurance. And data handling differs by plan and by whether you signed in with ChatGPT or an API key, so if your problem is embargoed, check the settings before you paste it in rather than after.

----------

## 12. Where this doesn't help

The sweet spot is a problem where you have real intuition, some finite structure to compute with, and more ideas than time to test them. That covers a lot of combinatorics, algorithms, and discrete optimisation.

----------

## Reference

```
codex                        start in the current directory
codex --sandbox read-only    look, don't touch
codex resume --last          reopen the most recent session
codex exec "..."             one non-interactive run, for scripts and sweeps

```

```toml
# ~/.codex/config.toml
model = "gpt-5.6-sol"
model_reasoning_effort = "xhigh"
review_model = "gpt-5.6-sol"

[sandbox_workspace_write]
network_access = true            # only if you need pip

[agents]
max_concurrent_threads_per_session = 8
default_subagent_model = "gpt-5.6-luna"
default_subagent_reasoning_effort = "high"

[features.rollout_budget]
enabled = true
limit_tokens = 2_000_000

[features.context_management]
experimental_mode = true         # Astra, Plus/Pro, ChatGPT sign-in

```

Official docs: [CLI](https://learn.chatgpt.com/docs/codex/cli) · [models](https://learn.chatgpt.com/docs/models) · [pricing](https://learn.chatgpt.com/docs/pricing) · [subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents) · [commands](https://learn.chatgpt.com/docs/developer-commands) · [sandboxing](https://learn.chatgpt.com/docs/sandboxing) · [config reference](https://learn.chatgpt.com/docs/config-file/config-reference) · [AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)

Codex moves quickly. If something here behaves differently, check `/status` and the changelog.

_Corrections welcome._
