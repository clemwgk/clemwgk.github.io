---
layout: post
title: (WIP post) Two Days to MVP - Building a Reminder Bot with Claude Code
---

Note: This post is partially drafted by Claude Code.

Developments in the AI/LLM space have been moving fast. Since I first started working on my email digest, there have been a lot more advanced, agentic tools available to us. Claude Code in particular has been hugely raved about, and so I found myself giving it a try with this reminder bot project.

## What's the problem?

First, let's set the problem stage. I wanted to lower the barrier to setting reminders. Why? At first glance, this is a solved problem, because there are many avenues through which I can set reminders, such as on Telegram (scheduled messages to self), or calendars on my phone. 

But existing methods all involve me having to decide when I want these reminders to fire, AND I still need to mechanically set the timing, create the reminder myself, and such. And there's more to how I use reminders (and hopefully it's not just me) - what happens if I want to delay or snooze a reminder? What if I want to set a recurring reminder? What if for certain types of reminders, I know I want multiple reminders? What if I want to cancel a reminder? Heck, is there a way I can check if I've already set a reminder for a task? Right now, any of these things require me to go back to where I've set the original reminder and make the edits myself. I could do it myself, but it feels more effortful to manage than I'd like.

Wouldn't it be great if a bot could help manage those things for me? 

The Minimum Viable Product (MVP) version is: I tell the bot to remind me to do something at when, and that "when" dictates when the bot sends the reminder to me. So I still have to spell it out, but I don't have to set the reminder myself.

In a well-developed version (we're not there yet), I think it would look like this: I send a message to the bot with the reminder text, the bot is able to parse the reminder text, classify it, then apply a schedule of when the reminders would be sent based on the type. 

I'm probably only a bit further along from the MVP version, but I'd like to think that the reminder bot I've got has already been helpful for me. Check it out [here](https://github.com/clemwgk/reminder-system-claude/blob/claude/automate-reminders-LInTS/README.md)

I consider myself comfortable to learn about technical things, but ultimately I'm not a software engineer, so writing a complex script to run the bot, setting the "infrastructure" to intake, process, and output the reminder, all that is stuff I'm not familiar with. Claude Code was really helpful here, and it's crazy to me that in about two days, I had a working MVP deployed on Google Cloud's free tier, reliably parsing natural language reminders and sending them to my wife and I. 

As with the LLM email digest, what I learned along the way was less about the code itself and more about how to work *with* an AI to build something.

## Prerequisites - Claude Code?

I'm not sure that it _had_ to be in Claude Code. But I did it in Claude Code because I have a Pro subscription (at the time of writing). It's possible that Codex (OpenAI's version of Claude Code, to my understanding) could do something similar too, but I didn't try it in Codex.

Honestly, that's all I think I needed. Claude Code guided me through the rest in terms of setting up the infrastructure (e.g. Google Cloud account).

By the way, I've also been experimenting with using Claude Code for non-technical tasks. It's been really great there too, and even Claude in browser hasn't been half bad - yeah I might be able to do it quicker myself, but I could give it a task to work on in the background while I do something in parallel. 

My experiences with ChatGPT and Cluade so far - especially the recent ones with Claude Code and the in-browser Claude - make me wonder if I can get access to "vanilla" Claude/Claude Code at work. It feels to me like a lot of wasted potential that comes down to permissions and access issues - we have these powerful tools but we can't fully benefit unless they can interact with the other tools we use at work? Anyway that's a different conversation I guess.

## What I built using Claude Code

A Telegram reminder bot that:
- Accepts natural language input ("remind me to pay the electricity bill next Saturday")
- Parses the intent using a Gemini LLM (2.5-flash-lite, free tier)
- Stores reminders in a local SQLite database
- Sends notifications when they're due
- Supports shared reminders ("remind us to call the plumber")

The tech stack: Python, Telegram Bot API, SQLite, Gemini for natural language parsing. Self-hosted on a Google Cloud free tier VM.

## The architecture

Claude Code sketched this diagram early in the conversation, and honestly, this was one of the most helpful things. As a non-engineer, having a visual map of what we were building made everything else easier to follow.

```
┌─────────────────────────────────────────┐
│  You + Partner (Telegram on phones)     │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│       Telegram Bot (Python)             │
│                                         │
│  - Intake: natural language → LLM parse │
│  - Management: /list, /delay, /cancel,  │
│                /snooze, /edit, /copy    │
│  - Delivery: sends reminders when due   │
└───────┬─────────────┬─────────────┬─────┘
        │             │             │
        ▼             ▼             ▼
┌───────────┐  ┌───────────┐  ┌───────────┐
│  Gemini   │  │  SQLite   │  │  config.  │
│   API     │  │    DB     │  │   yaml    │
│(for parse)│  │           │  │           │
└───────────┘  └───────────┘  └───────────┘
```

## The workflow

Here's what the feedback loop looked like in practice:

```
Me: [describes problem or shares screenshot]
     ↓
Claude Code: [diagnoses, proposes fix with tradeoffs]
     ↓
Me: [refines based on domain knowledge]
     ↓
Claude Code: [implements, commits, provides deploy command]
     ↓
Me: git pull && sudo systemctl restart reminder-bot
```

This cycle repeated dozens of times. What made it work:

1. **Clear instructions**: Claude Code gave me copy-paste commands for deployment, systemd service setup, and config structure. I didn't have to figure out the boilerplate.

2. **Iterative debugging with screenshots**: I'd share a screenshot of a bug, Claude Code would diagnose and propose a fix, I'd refine based on what I knew about our actual usage.

3. **Constraints were respected**: I told Claude Code "free tier only" and we ended up with Gemini 2.5-flash-lite (1000 requests/day). I said "privacy-conscious" and we went with self-hosted SQLite, no third-party logging. "Mobile-first" led to shortened UI elements after I showed real device screenshots.

## The interesting part: when I had to push back

The bot didn't come out perfect on the first try. Some of the most valuable moments were when I had to correct Claude Code's approach.

### Example 1: Defining cleaner logic

Claude Code proposed a `/snooze` command behavior that was... confusing. If you replied to a message about reminder ID 53 and typed `/snooze 48 10`, the initial implementation would:
- Extract ID 53 from the replied message
- Interpret 48 as the minutes argument
- Ignore 10
- Result: snooze reminder 53 for 48 minutes

That's not right. I proposed cleaner logic:
- If there are two numerical arguments, they're always {id, minutes} — regardless of whether you replied to a message
- If there's only one argument and it's a direct reply, the replied message provides the ID and the single argument is the duration
- If there's only one argument without a direct reply, we don't have enough info — error out

This explicit rule was better than Claude Code's initial attempt. The AI accepted the correction and implemented it.

[SCREENSHOT: Exchange 2 - /snooze design pushback]

### Example 2: Reframing the problem entirely

We hit a bug where "Remind me Friday if go back Florence to bring back the breastfeeding milk" was scheduled for Wednesday instead of Friday. Claude Code proposed adding keyword fallback for day-of-week detection.

I pushed back: that's the wrong approach. The real issue was that the LLM was confused by Singaporean English grammar patterns — preposition omission ("go Florence" instead of "go to Florence"), subject dropping ("if go back" instead of "if I go back").

The fix wasn't deterministic day detection. It was adding linguistic context to the LLM prompt explaining Singaporean English patterns. More generalizable, addresses the root cause.

[SCREENSHOT: Exchange 3 - Singlish reframing]

And then I had to refine further: the users (my partner and I) don't actually speak Singlish with particles like "lah" or "leh". We speak closer to standard Singaporean English — grammatical quirks but no particles. Claude Code had researched Singlish grammar and drafted a comprehensive prompt, but I knew our context better.

## What I learned

**1. AI as collaborator, not replacement.** The best solutions came from combining AI's technical capabilities with my domain knowledge. Claude Code could write the code, but I knew what "two arguments" should mean for `/snooze`, and I knew what kind of English we actually speak.

**2. Iterate with real usage.** The first solution is rarely the best. Bugs I caught from actually using the bot on my phone (like formatting that looked fine on desktop but broke on mobile) were things Claude Code couldn't anticipate.

**3. Know when to use LLM vs deterministic logic.** We used the LLM for fuzzy parsing — turning "remind me tmr to buy groceries" into structured data. But for commands like "cancel" and "list", we added deterministic keyword fallback before the LLM call. And for precise control, we used slash commands (`/copy` instead of letting the LLM guess when to duplicate).

**4. Context matters.** The Singaporean English fix was more valuable than a generic "day detection" hack because it addressed the actual user base.

## Features built

| Command | What it does |
|---------|--------------|
| `/list` | Show pending reminders |
| `/cancel <id>` | Cancel one or more reminders |
| `/delay <id> <hours>` | Delay a reminder |
| `/snooze <id> [mins]` | Snooze (default 15 mins) |
| `/edit <id> <text>` | Edit reminder text |
| `/copy <id> <time>` | Copy reminder to new time |

Natural language: "remind me to buy groceries tomorrow", "remind us to call the plumber" (shared), time shorthands like "tmr" and "nxt wk".

UX: inline buttons for quick snooze/done actions, direct reply support for all commands, Singaporean English grammar support.

## Closing thoughts

I went in not knowing Python and came out with a deployed, working reminder bot that my partner and I actually use daily. Claude Code did the heavy lifting on the code, but the moments where I had to push back or refine — those were when the real learning happened.

The workflow of "describe problem → AI proposes → I refine → AI implements → test on actual device" is powerful. It's not magic, and the AI isn't always right. But for someone learning to build software, having a collaborator that can write the code while you focus on the logic and the domain knowledge — that's a real accelerant.
