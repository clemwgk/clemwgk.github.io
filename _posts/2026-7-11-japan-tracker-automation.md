---
layout: post
title: "Automating Travel Research: The LLM Was the Easy Part"
---

Japan is one of my favourite holiday destinations that I return to; as I like to say, the best way to end a Japan trip is to start planning for the next one, and each time I try to add something new to the itinerary or explore a new region. One great source of information for me is an SG-JP travels telegram group chat in which many more well-travelled Singaporeans share info about hotels, scenic/tourist spots, food places, etc. Globally, it also looks like Japan is becoming more popular as a holiday destination too - maybe it's the weak JPY or is it just the social media algorithm figuring me out?

In any case, this means I often come across useful Japan travel leads, and I'd love to keep them filed away somewhere. Over time, that repository would become a handy source of "vetted" leads to reference later.

But it's tedious to manually file away leads as and when I come across them. Sources are varied, and any entry into a repository has to contain enough first-pass detail to make it actionable later on. For example, a hotel name with no location is not a great entry - I don't need to know _everything_ about that hotel, but minimally I need to know which part of Japan that hotel is in so that I can see if it's even in the region that I plan to visit. The problem is that capturing with enough first-pass detail, in a place that I could find it all later, is honestly onerous enough that I wouldn't bother to do it manually. If I could lighten the lift required to sustain/upkeep this repository, then that would be a useful workflow with a tangible benefit for me.

So I got Opus in Claude Code to build an automated capture and research workflow that makes it easy for me to store these leads, with sufficient first-pass detail.  

TL;DR: The automated workflow I stood up requires me to only do the step of forwarding the message or recommendation, or typing a light description into a Telegram channel that serves as the intake funnel, and everything else is automated. This post covers how I built that automation with Claude Code, what broke, and what I learned when I rebuilt it.

## Constraints shape the design

Before jumping straight to solutioning, it's good practice to think about constraints, because they shape the design of the solution. Constraints can be environmental, or can be strong preferences (strong enough to treat them as constraints), but either way, they narrow the set of options which is helpful for steering Claude. Claude would be able to evaluate what's technically possible or not, but you have to share your own constraints too.

I had these constraints in mind:
1. Zero incremental cost beyond my existing Claude Pro and ChatGPT subscriptions
2. No always-on infrastructure (my laptop is not always on)
3. Full automation after I type it into the intake avenue
4. Data goes somewhere I can actually plan trips from

I gave the problem and the constraints to Opus, and here are the key design choices we landed on, and how they connected back to my constraints.

**Private Telegram channel as the inbox.** Leads arrive in Telegram chats, and forwarding to a telegram channel is two taps on the phone. Telegram is my most-used messaging app anyway, and using Telegram makes it mobile-friendly. Alternative might have been something like using Gmail, which has more friction.

**Cloud agent over local cron.** Specifically, using Claude Code's cloud routines which run on Anthropic's infrastructure, not my laptop. This is included in my monthly Claude subscription, so no incremental cost. In contrast, if I set up a cron job on my laptop, that only works when my laptop is on.

**Connecting Telegram to Claude Code with polling over webhooks.** Telegram's Bot API offers two ways to get messages: `getUpdates` (you check Telegram for updates) or webhooks (Telegram pushes updates to call you). `getUpdates` is superior for my constraints because webhooks need a server to receive POST requests, which means something to deploy and maintain, whereas `getUpdates` doesn't require any infrastructure. But `getUpdates` only retains messages for about 24 hours, so the Claude Code agent needs to be run at least on a daily cadence to catch the messages before they expire.

**Airtable over Google Sheets.** This one was supposed to be simple. I usually plan using Google sheets, and Claude has a connector to Google, so that's clearly the natural home for the repository of leads, and was the original design. But it turned out that the Claude Code web Google MCP connector (which is Anthropic's first-party MCP connector) is read-only, so it can't write to Google sheets, i.e. Google sheets cannot be the repository host. Whereas the MCP connectors I use on my laptop's Claude Code CLI have the write access needed, but is local and cannot be called when running a Claude Code web routine using Anthropic's cloud environment.

So Airtable it is. It's free, cloud-reachable, and write-capable. I'd never heard of Airtable before this, but when I created the account, I guess it looked fine - it looks tabular and typed, kind of a familiar environment for me.

**Three-tier research depth.** The automation applies different research depths by category, because I don't think everything requires the same effort. Hotels need deeper research because I care about things like, does it have an in-room onsen, or do they allow children. Scenic spots just need seasonal timing (is there a best season to visit?) and access info. Food needs almost nothing. To be clear though, I think even a simpler research depth for hotels would have been fine, as long as I had enough details (location!) for first-pass filter for subsequent planning ease.

## v1 of the automated pipeline (works!)
Putting all that together, here's what v1 looked like:

```
[Telegram Channel]
    | (Me: forward a lead, 2 taps)
    v
[Cloud Routine, daily 08:00 SGT]
    | Claude Code: getUpdates -> classify -> research -> write
    v
[Airtable: Hotels / Scenic / Food / RunLog]
```

This design worked! I got this to run successfully on 1 Jun 2026, from my perspective it was all very simple for me. I dump leads into the Telegram channel, and Claude Code runs daily at 8am SGT to process my leads.

Okay, but what about monitoring though? As I learnt from a [prior automation attempt](/substack-summary-failure-postmortem/), monitoring is really important to ensure the automation is working as expected. The dangerous failure in an automated pipeline is the silent one. So I was deliberate about wanting stronger monitoring this time.

v1 had four monitoring layers: per-lead offset commit (only advance after a confirmed write, so a crash mid-batch can't silently drop a lead), a RunLog audit trail, push notifications on adds or errors, and a health dashboard for schedule liveness and failed research (though to be honest I haven't really used this one). When the first cloud run attempt failed (Telegram API blocked by the network; I needed to add Telegram to the environment allowlist), it sent a push notification to notify me. So silent failure shouldn't be a problem.

![Push notification alerting me that the first cloud run failed](/2026-07-japan-tracker-automation/push-notification-run-failure.jpg)

But silent failure is not the only failure mode, and as you'll see below, v1 monitoring had a structural blind spot: **it relied entirely on the LLM run to self-report.** The RunLog, the notifications, the offset commits were all written by the same LLM routine that did the processing. If the routine ran and saw nothing (`getUpdates` returned empty), it faithfully reported "0 messages found" and moved on. There was no independent record of what Telegram actually returned, no raw forensic trail, no way to distinguish "nothing was there" from "something was there but got missed." The monitoring could tell you when the system crashed, but it probably couldn't identify other, more subtle failure modes.

## The Jun 16 miss

On 2026-06-15 at around 18:20 SGT, I forwarded a message containing a lead for Prince Hotel Shinagawa. But the next two daily runs both returned zero messages, so there was no entry for Prince Hotel Shinagawa in the Airtable.

The daily routine itself was running though, so what was the issue? Well, I wasn't sure, so I thought I'd ask Claude Code. 

![My exchange with Claude Code trying to debug the miss, part 1](/2026-07-japan-tracker-automation/asked-claude-code-1.png)  
![My exchange with Claude Code trying to debug the miss, part 2](/2026-07-japan-tracker-automation/asked-claude-code-2.png)

That wasn't very helpful at all, but it seemed like re-forwarding it did the trick, so that means the miss was on the ingestion side, not a crashing routine. So this single miss surfaced two problems:

1. Ingestion was fragile. I forwarded the message around 6:20pm, which is comfortably within 24 hours when the next Claude Code run on Jun 16 came around. But for some reason it didn't work. And compounding the problem is that Telegram's `getUpdates` is an ephemeral, destructive, pull-once queue. So if the workflow failed once, the lead would almost certainly fall through the cracks thereafter — the next daily run comes ~24 hours later, past the retention window. Another plausible scenario that could lead to a lead falling through the cracks (not related to this miss) would be if I hit my weekly Claude Code limit, and the next scheduled Claude Code run doesn't execute.
2. Worse, the monitoring didn't help. Because the monitoring is all self-reported, there was no record of the raw `getUpdates` response from the runs that missed it. Three possibilities: the update genuinely never appeared in the bot's queue, it appeared but was skipped over, or it expired from Telegram's retention before the daily poll reached it. If the system had kept a raw log of every Telegram update it saw (or didn't see), I would know which one. Instead, I had a RunLog that said "0 messages found" and no way to challenge that.

There were already indications that the self-reporting was failing. The `run_timestamp` field was being written as midnight instead of the actual run time (confirmed as 8am SGT in the Claude Code web UI). The same LLM that was self-reporting "everything is fine" was also getting timestamps wrong. But I still couldn't _conclusively_ prove that the Claude Code instance missed it.

![The RunLog writing run_timestamp as midnight instead of the actual 8am run time](/2026-07-japan-tracker-automation/timestamp-drift.png)

## Designing v2

The Jun 16 miss showed that while v1 was a working process, it was also very brittle.

The essence of solving for the two problems surfaced by that Jun 16 miss was actually quite simple. LLMs are inherently non-deterministic systems, but some parts of the process were actually deterministic. Checking for the presence of new messages, logging the timestamp of the execution, updating the message offset... these were deterministic parts of the process, there was no judgement work involved there. The LLM's real value-add was parsing the message to identify the subject (i.e. the actual hotel, food, or scenic view), and then doing the research to get the first-pass details. That was the non-deterministic part of the process.

What if we could separate the deterministic parts of the process out from the LLM's scope, so that the LLM doesn't have to deal with that? Determinism was the right track, but as I chatted more with Claude Code, it didn't seem to be exactly it either. Why? Because if we asked Claude to run a Python script for the deterministic steps, that's ultimately still subject to Claude's own self-reporting when it executes the daily processing.

A few back-and-forths with Claude later, we crystallized the core idea into four words: decouple capture from processing. The daily Claude Code run won't have to do it all at once.

The best way to picture this is a physical inbox tray. A mail-sorter (GitHub Actions) drops letters (Telegram messages containing the leads) into the tray every four hours. A researcher (Claude Code) works through the tray once a day. They never meet, never call each other. The tray decouples them. And that decoupling is precisely what lets a lead survive the processor being down for a day, or me hitting my weekly Claude Code limit, or whatever else might go wrong on the processing side. Once a message lands in the Inbox, it's safe. It can't be lost to Telegram's retention window anymore.

To make the design and implementation more robust, I also had Codex review Claude's proposed build plan to identify critical blockers/gaps (like getting a peer reviewer). As it turns out, you can have Claude Code call Codex directly in the CLI, but I didn't know that at the time and I did it manually by getting Claude Code to produce the design and build plan in markdown files and pointing Codex at them.

Here's what v2 looks like:

```
[Telegram Channel]
    | getUpdates (Bot API, pull)
    v
[LOGGER - GitHub Actions cron, every 4h, PURE PYTHON, no LLM]
    | writes raw updates -> Inbox table
    v
[Inbox table (Airtable)] -- durable, the source of truth
    | status = pending
    v
[PROCESSOR - LLM routine, daily 08:00 SGT]
    | reads Inbox -> classify -> research -> write
    v
[Hotels / Scenic / Food]  +  [RunLog]  +  [Health dashboard]
```

The key design choice: The message logger lives on GitHub. Running Python on GitHub Actions means free cron, hosted on GitHub, and no LLM involvement in the logging path at all. Met all my constraints. The alternative was to set up a second Claude Code routine that pulls a git repo and runs Python from there (i.e. split the existing Claude Code routine into two sub-routines), but this just felt like an inferior option when GitHub Actions provided a clear way to remove Claude Code from the deterministic steps.

Now that the process was split into the logging and processing cleanly, monitoring was improved too. The biggest change is that the Inbox populated by the logger becomes an audit trail that can immediately answer whether capturing the message was successful. Either way, that extra evidence helps narrow down the root cause. The RunLog also gets enriched with offset tracking, HTTP status, and a `possible_gap` flag that fires when an expected update ID is missing. If another miss like Jun 16's happened again, I could pull up the Inbox rows for that window and see exactly what `getUpdates` returned (or didn't).

There are a couple of other improvements too. In v1, the whole batch stopped on any failure, because the ephemeral queue made skipping unsafe (you might lose the skipped message forever). Now the raw message is durable in the Inbox, so the processor can mark a bad lead "error," retry it later, and keep going. And the two halves (logger and processor) watch each other through Airtable: the logger flags a stalled processor, and the processor flags a stalled logger. A dead component can no longer be invisible.

## Learnings and reflections

Even a seemingly simple automated workflow surfaced issues. Here are my key takeaways:

**Bring the constraints; let the LLM map the solution space.** The parts of this build I couldn't have done alone were the technical unknowns — that you could use `getUpdates` to pull messages from Telegram but has a 24 hour retention, that the Anthropic's Claude Code Google MCP connector is read-only, etc. The LLM could investigate and figure out these technical details for me, and I certainly would only have known some of these at best, if any at all. But the LLM couldn't have _decided_ for me either, because every choice hinged on my constraints and preferences — cost, no always-on laptop, somewhere I'd actually plan trips from. The build plan was the meeting point: I supplied the constraints, Claude supplied the technical map, and together we narrowed to a design neither of us had at the start.

**Probe assumptions before building on them.** And to reduce manual lift, ask LLMs to name and test the assumptions. Claude's initial plan for v1 said that "Scheduled agents run on Anthropic's cloud infrastructure. They have access to MCP tools (Google Docs, Gmail) and web search." But it turns out it was incorrect because the write-capable MCP was local on my laptop, and the cloud routine only has Anthropic's read-only MCP. In v2, I got Claude to probe the assumptions as much as possible. I don't think you would always be able to test all the assumptions, but I think it's good practice to try and do some of it at least (and get LLMs to do it to reduce the manual work), because if you do manage to find a problem at the pre-build stage, you just pre-empted potentially abortive work and saved your own usage limit.

**Choosing not to build something is a design choice.** After the Jun 16 miss, the `getUpdates` versus webhook decision came up again. The webhook did seem to be a good option to pursue to rule out any message retention problem on the Telegram side going forward, but the truth is that I was never able to prove whether that Jun 16 miss was a failure on the Telegram message retention side, or on the LLM processing side. So rather than go for the webhook fix now (which would immediately add a new account and maintained infrastructure), I went with the logger/Inbox fix, which could also solve/mitigate the problem, and still leaves me the option to pursue webhook in future. There was no reason to build the webhook path at this point.

**Use deterministic systems for deterministic work; reserve the model for judgment.** The timestamp drift showed that LLMs will improvise on fields that should be deterministic. Rather than ask it to do deterministic work and expect only perfection, it's safer to ringfence the deterministic steps as much as possible, so that the LLM can focus on the work that requires judgement. v2 draws the line explicitly. Timestamps, offset arithmetic, skip rules, and capture itself all live in tested Python. The LLM does classification, web research, and field extraction, which is the work that actually needs judgment.

I should caveat here: this might give the impression that I'm saying that LLMs should take over judgement-heavy pieces when trying to automate. That is true _for this workflow_ but I will not say so as a general rule of thumb. At the end of the day, you have to take into account the nature of the workflow and how complex this "judgement" piece is. In this case, the "judgement" was still fairly straightforward — identify the subject, collect some info to populate first-pass details, then record it in the table — so I think it's perfectly fine for 2026's LLMs to execute.

**The hardest problem is the plumbing, not the LLM's capabilities.** I don't know if this is a spicy take, but I found that every difficult problem in this project was system integration: how do I get Claude Code to ingest my messages in Telegram? can Claude Code on the web write to Google sheets? and so on. That's also consistent with my experiences trying to build and automate stuff in both my personal life and at work. 2026's LLMs are not short on brainpower for many tasks, the plumbing is just bad.

The LLM is the brain, but brains need arms and legs to be useful, first to ingest all the necessary context, and second to be able to act on your behalf. Some apps have out-of-the-box integrations that provide them. But many don't yet, and I believe that gap will continue to be the hard part of LLM-based automations for a while. The connective tissue between systems is what made this project difficult, not the AI. For a single-user workflow, I was able to chain together some managed primitives to make it work (Telegram's free bot API, Airtable's free tier, GitHub Actions' free cron, Claude Pro's included cloud routines), but I'm not sure how I would scale this to multiple users (not that I need to for this project).

The brain was never the hard part. The arms and legs were — and for now, wiring them up yourself is the price of admission.
