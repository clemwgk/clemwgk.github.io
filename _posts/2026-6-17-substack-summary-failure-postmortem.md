---
layout: post
title: Specs Aren't Enough: Reflecting On Failure Modes in Automated Workflows
---

One of the most helpful things in my AI learning journey has been the sheer abundance of AI-relevant material out there. Many of these articles come with linked prompts or guides that I'd want on hand for future reference, even if I don't need them today.

In February this year, I built a Claude Code skill, `/substack-summary`, that summarizes Substack articles and extracts their linked resources into Google Docs. It isn't on a schedule (Claude Code's routines feature didn't exist at the time), but I can trigger it whenever I come across an interesting article, capturing the resources instead of letting them get lost in my inbox. It lowers the overhead of saving these references enough that I'll actually do it.

The skill has one rule that matters above all others: linked resources get extracted **verbatim**, not summarized. These resources were often prompts written by the article author, and I wanted them word-for-word so I could study how the prompts were constructed. I ran the skill at least four times since then, and the only complaint I raised was about section ordering, not content quality. It looked clean. It wasn't.

This post is a retro on my learning lessons and what I took away. **TL;DR: "don't trust, verify"** (yes, the crypto ethos!). The resources weren't copied verbatim, despite the spec demanding exactly that. When we delegate to AI, we have to build the discipline to check whether the spec was followed, at the time it matters, before the evidence is gone. Evals and monitoring are the implementation of that discipline.

## What the skill does

The `/substack-summary` skill fetches a Substack article via a headless browser (Playwright), identifies the linked resources in the article (guides, prompt kits, tools), navigates to each one, extracts the full content, and produces a structured Google Doc. The intended output is an executive summary at the top, one section per fetched resource with its full verbatim text, then a resource index at the bottom.

The spec in the SKILL.md is explicit: "extract **full verbatim text content** — Do NOT summarize or synthesize.".

## The problems I saw

Running the skill at least four times surfaced a handful of problems. These were visible, and therefore fixable, even if I didn't always choose to fix them.

The biggest was section ordering. The first version of the skill produced the Google Doc backwards, with linked resources at the top and the executive summary at the bottom. The cause was a last-in-first-out bug: the helper script inserted each section at position 1 in the document body, so the last section ended up on top. This was the only quality complaint I raised in those sessions.

I think this is probably a small instance of a cascade failures, where one fault propagates into the next. A latent authoring error (the LLM that set up the skill hard-coded "always insert at position 1") sat dormant until a later instance dutifully ran the misconfigured helper, at which point it surfaced as a visibly broken doc. Context degradation could be the underlying root failure here: this was before 1M-context models, so the instance that wrote the skill may have lost track of how "always position 1" would interact with the rest of the skill instructions.

There were a few other problems that were fixable, but none painful enough to act on, so I lived with them for at least a few runs before asking Claude to update the skill.

- Tab API mismatch. I'd asked Claude to use native Google Docs tabs for each resource section. Every session, the tab creation call failed with HTTP 400 (not supported by the MCP), and every time it fell back to document sections instead, which was good enough for me.

- OAuth token expiration. Google revoked the refresh token three times over six weeks, and each time all Google Drive operations stopped until I re-authenticated manually. The steps were finicky, and only on the third occurrence did I finally ask Claude to document the recovery process in the skill and improve the error message, so it would surface actionable steps instead of a raw HTTP error.

- Argument list too long. The helper script's CLI wasn't built for large payloads, so passing a 42KB JSON via bash command substitution exceeded the OS command-line limit. Each time, Claude wrote a wrapper script to read the payload from a file instead.

## The failure I thought I caught

Here's where it gets more interesting. In one session in March, I noticed the output for a linked resource wasn't verbatim. The model had decided to condense the content, which is a serious failure (specification drift). In session logs I uncovered later while preparing this retro, Claude had apparently said it would produce "a comprehensive but not overly verbose version" of a guide it was told to copy word for word.

I caught it. I deleted the output doc and had the skill redo it. I skimmed the replacement, it looked fine, and I moved on.

Months later, when I went back to actually examine that replacement doc, I found four code blocks, each replaced with the literal string:

> Code available on the guide page — click "View & Copy Code"

![The replacement doc, with a button label transcribed in place of the actual code](/2026-06-substack-summary-postmortem/Screenshot%202026-06-17%20124648.png)

That's a button label from the source page! The code blocks were hidden behind collapsible elements that you had to click "View & Copy Code" to expand. For whatever reason, the skill copied the page as it saw it — without expanding the collapsible elements — and faithfully transcribed the button text in place of the actual code.

My read is that this is also specification drift, just from the other direction. Arguably, Claude did capture the text verbatim, but missed the intent of why I wanted to do it in the first place. Maybe if the intent was spelt out more explicitly or emphasized more in the skill, Claude would know to expand the text first before capturing the resource.

Either way, no error or warning was thrown. The output doc looked structurally complete. I'd already caught one verbatim violation on this same article and remade the doc, so I didn't expect a problem with the replacement. That was the trap: catching the first violation gave me false confidence in the redo, which I had no real basis for.

## Artifacts, Evals, and Monitoring

The irony is that the failure I missed is the one I can prove. Because I never caught it until I started this retro, I can open the replacement today and find the failure frozen in the artifact. The evidence survived because it's in the output, even though it went unnoticed for months.

The failure I caught is the one I can't prove anymore; I can only recount it from memory. I deleted the original output doc when I remade it, and the raw March session logs have been pruned from my laptop by Claude Code's retention policy. The source page itself was rewritten between the run and my review. I have the model's stated intent to condense and my memory of catching it, but nothing else. The evidence eroded in the weeks between the session and the retro.

I also considered that maybe the model didn't fail at all, and that the skill instruction _at that time_ simply didn't specify verbatim extraction. But when Claude couldn't find any indication in the skill change history or session logs that the skill instruction had changed around then.

**What are my learning points here?**

Specification drift is real. An explicit spec wasn't enough on its own. The system still shipped non-verbatim output twice on the same article, and the second time went undetected. Having the right instruction isn't the same as having the right output.

So how do we address it? The constraint is that ground truth has a shelf life. The crux is setting up the right evals and monitoring from the start, so we can measure and detect failures **at the time they occur**. Even if I had kept the original output doc, I couldn't run a line-by-line comparison against the source today, because the source has since changed.

Evaluation and monitoring at (or near) the moment of the run is key. In my view they aren't the same thing, but both matter: evaluation asks "is the output right?", and monitoring asks "is the process broken?".

For an ad-hoc, manually triggered skill, eval is probably the more important piece. If I were rebuilding this skill today, I'd add a post-run verification step, preferably something deterministic that an agent could run via a pre-written Python script. Even something lightweight would help: flag sections that are suspiciously short, check for placeholder text like "click here" or "view code," compare the section count against the number of resources identified. The replacement doc would have failed a placeholder-text check immediately. I just didn't have one.

I would also want real-time logging instead of after-the-fact reconstruction. A structured log at the moment of the run would have captured what was fetched, what was written, and whether they matched, before the source page got rewritten and the session logs got pruned.

The good news is that today's LLMs also enable better agentic QA. The best QA will always be the human, but having an agent spin up subagents to QA its own work is a reasonable compromise, and I imagine that's the way forward for low-stakes tasks, reserving human QA for the highest stakes. I can see myself doing exactly that: if I spend my own time QA-ing the workflow on every run, did the automation really save me any bandwidth? The point is to tier the QA and check the output _selectively_.

For a routinely scheduled automation, monitoring becomes equally important. How do we even know the process ran at all? The process needs to be suitably noisy, so that failures are noisy too, rather than silent. I recently started trialing a script to auto-archive and delete Gmail, and I have it email me after each run as a kind of "proof of work". Failures in the process would be "noisy" (visible) - the moment that email says nothing happened, or doesn't arrive on schedule at all.

Stepping back, the core lesson is a mindset one: when we delegate to AI, we have to build **the discipline** to check whether the spec was followed, at the time it matters, before the evidence is gone. Evals and monitoring are just the implementation of that discipline.
