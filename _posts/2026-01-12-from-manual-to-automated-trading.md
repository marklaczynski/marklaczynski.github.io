---
title: "From Manual Trading to Automated System: Why I Built Brainscan"
date: 2026-01-12 14:30:00 -0500
categories: [PROJECTS, TRADING]
tags: [trading, automation, options-trading, portfolio-management, algotrading]
---

# From Manual Trading to Automated System: Why I Built Brainscan

## The Consistency Gap

After years of trading, I noticed something uncomfortable: I was incredibly inconsistent with my trades. I'd sit at my platform, staring at all the numbers and lights flashing on charts, and I'd over-analyze. Overthink. Second-guess.

I tried to follow the tastytrade mechanics—the 50% profit rule, rolling at certain points. But after years of trading, I realized I couldn't even tell you if I was actually closing at 50% or just winging it based on how I felt that day. Part of the problem was that even when I thought I had a plan, it was half-baked. I'd tell myself, "Yeah, I have this 50% rule, I'll roll or close at that point." But then what? Do I close the position? Roll it up? Roll it out? Under what circumstances?

I hadn't defined all of that. And that lack of definition ended up biting me in the ass because I'd make decisions on the fly. Without a step-by-step procedure, even my "rules" became gut-feel decisions in the moment.

## The Double Problem

### The Emotion Tax

I thought I had a plan. But even when I had a trigger—say, 50% profit—actually executing on it was difficult. I'd usually default to just closing the trade because I'd think, "Well, the market has already moved in my favor. I can't roll up now, it's probably going to reverse." That kind of fear-based reasoning wasn't helpful, but it felt rational in the moment.

And when a position went against me? I had no clear system. Do I roll to the same strike? Roll down to a 30 delta strike? Under what conditions would I do either? I didn't know. So I'd second-guess myself constantly.

The emotional problem wasn't just about making bad decisions—it was about feeling like I wasn't trading well at all. I was so bogged down and weighed down by trying to get the mechanics working that I never felt confident. I wasn't executing a full plan because I didn't actually have one.

### The Time Trap

I spent way too much time on the platform. Way more than I needed to.

I'd log in, look through each of my positions, do very mechanical things—check the profit percentage, check days to expiration for when to roll to the next month. Rote activities that didn't require any real cognitive capacity. But then I'd also start looking at what the market was doing in general, and I'd fall into rabbit holes. "Oh, this stock moved today, let me research it. Maybe I'll put a position on it because I just sort of feel like it."

Or I'd start looking at charts for stocks in my portfolio, and I'd get an emotional response from that. "The stock has gone up, maybe I should add a position here." The platform, in a way, distracts you from what you really need to be focused on. There are high-value tasks you can do, but I wasn't doing them—I was wasting time on low-value busywork.

I'd log in probably a few times a day, spend 20-30 minutes each time. So I'd waste a good hour or two daily just sitting on the platform, half the time not even doing anything meaningful. Because it's not like every single moment of every single day needs some activity.

## The Realization: I Was Thinking Wrong

The first time I really learned that portfolio-level metrics matter was when Tom Sosnoff explained it, probably around 2012-2014 when I first started watching tastytrade. He said something like: you have a bunch of positions, they're all moving up and down, and you just ignore all of them except the one that's at your target profit. That's the one that needs action. Otherwise, you just look at your portfolio Greeks.

It made sense to me. Logical sense. But I never felt like I was actually free of the details of each position to really put mental energy into the portfolio-level metrics. I was too bogged down in the weeds.

I realized the only way I'd actually think this way was if I had no choice. I needed to build a system that would force me out of the position-level weeds and make me think portfolio-first from the start. That's when I decided to build Brainscan.

## Enter Brainscan: Automation Meets Portfolio Intelligence

### The Core Philosophy

The core philosophy of Brainscan is full automation. Something that requires little to ideally zero human intervention. That's still a goal I'm working towards, but the human intervention is currently at a very minimal amount compared to how much time I was spending on the platform before.

I also made a deliberate choice to start as simple as possible: single-leg options only. No complex multi-leg strategies to start. I wanted to nail the fundamentals—automated position management, profit-taking, rolling—before adding complexity. Multi-leg support is planned for the future, but keeping the initial scope tight let me focus on getting the core mechanics right.

Full automation, but with solid logging so I could understand why each trade was being made. I wanted to follow the sequence of events leading up to a trade so I could audit everything that happened. This also helped with debugging both the application and the trading strategy itself.

The other key piece was rule-based signals. Codifying all the events and all the decisions that would happen because of an event, based on context and rules. I wanted it to be fully automated, running in the cloud, hands-off. Not even a user interface. I basically only wanted to make changes to the code when needed—like, "I think I'm going to adjust this piece." Eventually a UI to help with configurations would be nice, but that's only to manage configuration parameters. Not the trading strategy itself. The trading strategy should just run on its own based on the parameters I set and configure.

And it worked. Once the system was running, I experienced that shift I'd been looking for. I wasn't tweaking positions anymore—I was tweaking code. I stopped obsessing over individual position deltas and P&L, and finally started thinking like a portfolio manager.

### How Each Component Solves a Problem

#### 1. Signal System → Consistency Without Emotion

The system evaluates rules I've set up for my trading and generates signals automatically. Those signals are logged so I can see exactly what's going on.

For example, I have a rule around taking profits at 50%. When a position hits that threshold, the system triggers a signal, and that signal generates an actual trade. Now I don't have to log into the platform—the code automatically sends that trade over to the broker and executes it for me.

No second-guessing. No "should I wait for more profit?" The rule fires, the signal executes. The same logic evaluates every time, consistently.

#### 2. Delta Optimization → Portfolio-Level Thinking

After all my trades are sent over, I do an end-of-day portfolio-level decision: adjust my deltas. This hedges me against directional risk beyond a certain threshold I'm comfortable with.

This lets me think at the portfolio level instead of just buying or selling futures on a whim, trying to scalp them like I used to. I actually use them for real hedging against my portfolio now. I think I'm using them the way they were intended to be used—probably the best use for them instead of trying to make short-term day-trading trades with futures just because they're leveraged.

I use futures now for their capital efficiency in hedging my portfolio. I basically don't even consider individual position deltas—probably not at all. I only look at the portfolio-level delta when I make decisions to hedge. And I make sure to beta-weight them to the product I'm using to hedge.

#### 3. BPR-Based Position Opening → Systematic Growth

Codifying my strategy forced me to figure out how to size my opening positions. Before, I never really had a true rule about this. I was never consistent with it.

With this application, I was able to define what my opening size would be based on the size of my net liquidation value. And I used buying power reduction (BPR) as a strict measure of how much capital I would be deploying. This lets me scale how much risk I'm putting on based on my portfolio.

I manage my buying power by looking at the IV rank in the QQQs—the Nasdaq. If the IV there is low, I deploy the minimum amount of positions to fill up the minimum BPR. Then I scale that up as the VIX and IV rank increases, deploying more capital and positions as that goes up.

#### 4. Performance Tracking → Data-Driven Improvement

One of the most recent things I've implemented was performance tracking so I could do data-driven improvements. What that basically means is I log all my portfolio performance, and I can identify when I made a code change and how that may have impacted returns over time.

This creates an audit trail linking changes to portfolio performance. That's really important because you need to know how a configuration change or a code change performs over time, and which change made what effect. Then you can test your code changes or configuration changes and see whether those yield higher results or worse results.

The drawback is that it's on a live single system—there's no A/B testing because I don't have multiple portfolios. But that would be an ideal thing to work towards one day.

## The Results: Freedom & Discipline

The results have been amazing. I still monitor my account through the platform, but I no longer spend any time deciding if I should make a trade. I just see what's going on, make sure the system isn't doing anything crazy.

I basically just check in daily, usually towards the end of the day, to see what trades my system has sent over and what's gotten executed. It's been phenomenal because it's saved me the cognitive load of making decisions and removed all the emotion from it. And being able to better track my performance with my internal tooling is really powerful.

I've saved time and cognitive load—those are probably my two biggest wins. I can just trust the system now to do its thing. And I spend my time thinking about how I can adjust the system to try to yield better returns.

## Technical Implementation (Brief)

I implemented this in Python. The biggest driver for that decision was that there was already a tastytrade SDK written in Python that wrapped the tastytrade API for me. It's actually pretty good—saved me a lot of time not having to deal with the API directly.

Python is a pretty easy language to use. It is a little slow, which is the downside, but considering this thing isn't a real-time engine yet (maybe in the future it will be, and I'll have to rewrite it), it's good enough for what I need to do.

Every time it spins up, it does real-time price streaming. It runs on a schedule on AWS Lambda—triggered at specific times during market hours rather than running continuously. I use CloudWatch for my logs.

I've also written basically the whole thing using Anthropic's Claude models. I've noticed that as the models have gotten better over the last year, and the tooling has improved, my coding has become way more proficient. The code quality has gotten significantly better. I use Claude Code on the terminal to develop, with a bit of VS Code for anything I need to do manually.

## Lessons & What's Next

The transformation I mentioned earlier—thinking at the portfolio level rather than the position level—has been profound. I see my positions, but I don't tend to care about them. I'm quite disconnected from my individual positions, actually. I couldn't care less if I have a position that's losing money or down on the year. My only goal is to get my overall P&L to be positive and moving in the right direction.

### What's Next

I have lots of plans for enhancements:

- **UI for configuration**: Could be useful for managing parameters without touching code
- **Hedging strategies**: Experimenting with different approaches and frequencies (daily vs weekly cadence)
- **Strategy tweaks**: Different criteria for rolling or exiting positions, continuous improvement
- **Multi-leg options**: As I mentioned earlier, I started simple with single-leg options only. I think there's value in expanding to multi-leg positions in the future.
- **Performance analysis tooling**: Right now it's very rudimentary. I have the raw data, and the tooling's barely there—just enough to give me some insight. I want to build it up to be more user-friendly and provide deeper insights.

## Closing

Remember when I said I couldn't tell you if I was actually closing at 50% or just winging it based on how I felt that day? Now when a position hits 50% profit, the same rules execute. Every time. The system evaluates the exact same logic I've defined—no gut feelings, no second-guessing, just consistent execution of my predetermined strategy.

But the transformation went deeper than just consistency. Forcing myself to codify every single decision—having a reason for it, or at least a thesis—made the system powerful and helped me grow as a trader. I went from being a position manager to a portfolio strategist.

Trading is now a consistent, mechanical process that doesn't require my constant attention. That's freed me up to do much more creative work—tweaking the system, exploring new strategies, thinking strategically rather than tactically. That's not just efficiency. That's freedom.

