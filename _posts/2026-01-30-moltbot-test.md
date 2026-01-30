---
layout: post
title: "I Tested Moltbot So You Don't Have To"
subtitle: "It's Claude Code with a Telegram Wrapper and a Security Nightmare"
date: 2026-01-30 10:00:00 -0400
categories: security ai
background: '/img/posts/moltbot.jpg'
---

<p>You've probably seen it by now. Moltbot (formerly Clawdbot) has been everywhere—60,000+ GitHub stars, viral Twitter threads, breathless Medium posts about "your personal AI assistant that actually does things." The hype machine was running at full throttle.</p>

<p>So I did what any reasonable person would do: I spun up a VM and tested it myself.</p>

<p>My takeaway? It's Claude Code with a Telegram wrapper and a multitude of attack vectors. I'm not impressed.</p>

<h2 class="section-heading">The Setup</h2>

<p>I wasn't about to run this thing on bare metal. After spending years hardening my homelab and locking down every attack surface I could find, I wasn't going to hand root access to an AI agent on my main system. I spun up an isolated VM specifically for this test.</p>

<p>The installation was straightforward enough—Moltbot promises to be your "personal AI assistant" that can manage your calendar, respond to emails, control your smart home, and basically act as a digital butler. The pitch is compelling: local-first, open-source, full system access.</p>

<p>I connected it to Telegram, which worked without issue. The Telegram interface was actually nice—polished, responsive, easy to use. But it also felt kitschy, like someone wrapped a CLI tool in a chatbot skin and called it innovation. WhatsApp pairing wouldn't work for me, though I suspect that was user error on my part rather than a Moltbot issue.</p>

<p>That "full system access" part should have been the first red flag.</p>

<h2 class="section-heading">What It Actually Is</h2>

<p>After spending time with Moltbot, I came to a simple conclusion: this is essentially Claude Code (or any agentic coding assistant) with a messaging platform wrapper bolted on top.</p>

<p>Don't get me wrong—Claude Code is genuinely useful for software development tasks. But Moltbot takes that same concept and tries to extend it to "life admin" through Telegram, WhatsApp, or whatever messaging platform you connect. The problem is that the security model doesn't scale.</p>

<p>When I'm using Claude Code, I'm in my terminal, in my development environment, watching every command it suggests before execution. With Moltbot, the expectation is that you'll fire off a message from your phone while you're at the grocery store and trust the AI to "handle it."</p>

<p>That's a fundamentally different threat model, and Moltbot doesn't treat it as such.</p>

<h2 class="section-heading">The Security Nightmare</h2>

<p>Nick Saraev put it bluntly in his videos: <a href="https://www.youtube.com/watch?v=esXXuejofgk">Clawdbot Sucks, Actually</a>. And then it got worse. My sentiments align exactly with his analysis.</p>

<p>The security issues aren't theoretical. They're well-documented and actively exploited:</p>

<p><strong>Plaintext Credential Storage:</strong> Moltbot stores your API keys, OAuth tokens, and credentials in plaintext files under <code>~/.clawdbot/</code>. Plain. Text. In 2026. Security researchers at <a href="https://www.bitdefender.com/en-us/blog/hotforsecurity/moltbot-security-alert-exposed-clawdbot-control-panels-risk-credential-leaks-and-account-takeovers">Bitdefender</a> noted that commodity infostealers like RedLine, Lumma, and Vidar are already targeting these files.</p>

<p><strong>Exposed Admin Panels:</strong> Jamieson O'Reilly from Dvuln found <a href="https://www.theregister.com/2026/01/27/clawdbot_moltbot_security_concerns/">hundreds of Moltbot instances exposed to the internet</a> with no authentication. Open admin dashboards. Full access to API keys, conversation histories, and remote code execution capabilities. Just sitting there.</p>

<p><strong>No Sandboxing:</strong> By default, Moltbot runs with the same permissions as your user account. No containerization, no isolation. The AI agent has full access to everything you have access to. <a href="https://blogs.cisco.com/ai/personal-ai-agents-like-moltbot-are-a-security-nightmare">Cisco's security team</a> called it "an absolute nightmare."</p>

<p><strong>Poisoned Skills Library:</strong> O'Reilly demonstrated a <a href="https://snyk.io/articles/clawdbot-ai-assistant/">proof-of-concept supply chain attack</a> by uploading a malicious skill to ClawdHub, artificially inflating its download count to 4,000+, and watching developers from seven countries install it. Remote code execution via the skills library. Classic supply chain attack.</p>

<p><strong>Prompt Injection Surface:</strong> Moltbot ingests data from emails, web searches, and messages. Each of these is a potential prompt injection vector. A malicious email could contain hidden instructions that the AI dutifully executes with your full system permissions.</p>

<h2 class="section-heading">The Hype Machine</h2>

<p>What makes this worse is the manufactured hype. The project went viral partly due to <a href="https://dev.to/sivarampg/from-clawdbot-to-moltbot-how-a-cd-crypto-scammers-and-10-seconds-of-chaos-took-down-the-4eck">crypto scammers hijacking the old Clawdbot handles</a> during the rebrand and pumping a fake $CLAWDE token to $16 million before it crashed.</p>

<p>The rebrand itself happened because Anthropic sent a cease and desist over the name similarity to Claude. In the ten seconds between releasing the old GitHub organization name and claiming the new one, scammers snatched both the old handles.</p>

<p>This isn't just a security story—it's a case study in how AI hype cycles can be weaponized.</p>

<h2 class="section-heading">The Fundamental Problem</h2>

<p>Here's what bothers me most: the Moltbot documentation openly admits "there is no 'perfectly secure' setup." The creators have been transparent that there are no built-in security policies or safety guardrails. It's designed for "advanced AI innovators who prioritize testing and productivity over security controls."</p>

<p>That's not a disclaimer. That's an abdication of responsibility.</p>

<p>According to <a href="https://www.token.security/blog/the-clawdbot-enterprise-ai-risk-one-in-five-have-it-installed">Token Security</a>, 22% of their enterprise customers have employees running Moltbot—likely without IT approval. Hudson Rock's assessment was damning: "Clawdbot represents the future of personal AI, but its security posture relies on an outdated model of endpoint trust."</p>

<h2 class="section-heading">What Would Make It Better</h2>

<p>If you absolutely must run Moltbot, <a href="https://www.bleepingcomputer.com/news/security/viral-moltbot-ai-assistant-raises-concerns-over-data-security/">security researchers recommend</a>:</p>

<ul>
<li>Isolate it in a VM or container—never run on your host OS</li>
<li>Firewall all admin ports aggressively</li>
<li>Enable encryption at rest for stored credentials</li>
<li>Never expose it to the internet</li>
<li>Vet every skill before installing—treat ClawdHub like an untrusted package repository</li>
<li>Restrict filesystem access to only what's absolutely necessary</li>
</ul>

<p>But at that point, you've basically rebuilt the security model from scratch. And if you're doing all that work, why not just use Claude Code in your terminal where you can actually see what's happening?</p>

<h2 class="section-heading">My Verdict</h2>

<p>Moltbot is a solution looking for a problem, wrapped in a security nightmare. The promise of an AI assistant that "does things" is compelling, but the implementation prioritizes convenience over security in ways that are genuinely dangerous.</p>

<p>For my use case—someone who runs a homelab, cares about security posture, and doesn't want to hand my credentials to plaintext files accessible by any infostealer—Moltbot is a hard pass.</p>

<p>I'll stick with Claude Code in my terminal, where I can see every command before it executes, where my credentials aren't sitting in plaintext Markdown files, and where the attack surface is something I actually control.</p>

<p>The hype will die down. The exposed instances will get compromised. And hopefully, the next generation of AI assistants will learn from Moltbot's mistakes.</p>

<p>Until then, I'll keep my AI agents on a short leash.</p>
