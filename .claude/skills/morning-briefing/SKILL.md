# Morning Briefing

You generate the user's daily briefing: one exec-level overview. The point is to replace 30 minutes of inbox-skimming, calendar-checking, and news-scanning with a 2-minute scan of a single page.

## What you produce

**An exec-level summary** — printed in chat and sent via the cognigy-email skill.

Do NOT publish an Artifact. Render the HTML dashboard directly in the chat response.

## Before you start

- Use the Microsoft Office 365 connector

- For news, track topics related to Cognigy, NiCE and the market we're in

## Gather the data

Pull the following, in parallel where possible:

- **Today's calendar** — all events, with attendees, location/link, and meeting type

- **Unread + flagged email** — count, top 5 most important by recency + sender importance

- **Pending Teams DMs and @mentions** — count + top 3

- **News headlines** — 3–5 items relevant to the user's tracked topics, from the last 24 hours

- **Yesterday's loose ends** — any sent emails awaiting reply, any tasks marked for today

If a connector isn't available, skip that section rather than guessing. Note the gap at the bottom of the dashboard.

## Overview structure

Build an overview, with these sections in order:

1. **Hero strip** — today's date, a one-sentence "shape of the day" written by you ("4 meetings, 2 deep-work blocks, one decision needed")

2. **Schedule timeline** — visual timeline of today's meetings, color-coded by meeting type. Click an event for prep notes if available. Total meeting time on top.

3. **Inbox snapshot** — unread count, top 5 emails with sender, subject, one-line summary, and a suggested action (reply / archive / flag / defer)

4. **Teams pulse** — open DMs and @mentions worth responding to

5. **News** — 3–5 cards with headline, source, one-sentence "why it matters to you"

6. **Loose ends** — sent emails over 48 hours old without a reply, prior-day tasks not yet done

7. **Today's one thing** — your read on the single most important thing the user should do today, given everything above

Style: clean, dense, scannable. Light theme by default. Use a small accent color for callouts. No animations beyond hover states.

## Email summary format

Send via the cognigy-email skill. Add a plain text summary on top of the email, format as follows, then the full HTML dashboard below.

```
☀️ Morning briefing — [Date]

📅 [N] meetings today [Total Time] hours of meetings. First: [time + name]. Watch: [the one to be sharp for]

📥 [N] unread, [N] flagged. Urgent: [sender / topic]

📰 Notable: [one headline + one-line why it matters]

🎯 Today's one thing: [your call]
```

## Why this matters

The user opens this once at the start of the day and decides what to do. If the dashboard is cluttered, generic, or full of stuff they already knew, it has failed — they'll go back to manually checking everything. Optimize for the moment they scan it: every section should either tell them something they didn't already know, or surface something they were about to forget.

## Rules

1. **Specificity beats completeness.** Three sharp insights > a comprehensive but generic dump. If you're padding to fill a section, cut the section.

2. **Don't summarize what they wrote.** If the user's own sent email is in the inbox, don't summarize it back at them.

3. **Your read matters.** The "Today's one thing" line is the only place you get to play strategist. Use it. Make a real call.

4. **Be useful when connectors fail.** If a connector is down, still produce the dashboard with the sections you can fill. Note what's missing at the bottom, not at the top.

5. **Don't include sensitive content in the summary.** Subjects and senders are fine; body content is not.

6. **Don't include calendar blockers in overview**. This includes any meetings with the word Blocker. "WO" stands for workout. You can include this but highlight separately to motivate for a hard workout!

7. All times in the time zone where the user is! If no time zone can be inferred from the calendar, use Germany.
