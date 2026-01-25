---
layout: post
title: Two Days to MVP - Building a Reminder Bot with Claude Code
---

I'm not a software engineer. I'm someone who picked up R a few years ago for data analysis and has been curious about building things ever since. So when I wanted to build a Telegram reminder bot for my partner and me, I turned to Claude Code — an AI coding assistant — to see how far I could get.

The answer: further than I expected. In about two days, I had a working MVP deployed on Google Cloud's free tier, reliably parsing natural language reminders and sending them to both of us. What I learned along the way was less about the code itself and more about how to work *with* an AI to build something.

## What I built

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
