---
layout: post
title: "host-sentry: A Local Security Monitor for the Machine Running Claude Code"
subtitle: "When your laptop is the canary in the coal mine, you should probably be watching it"
date: 2026-04-11 08:56:00 -0400
categories: security tools homelab claude
background: '/img/posts/claudesecurity.png'
---

<p>I use Claude Code for a lot of things — building bots, forensics tooling, homelab automation, troubleshooting things that would take me three times as long to solve on my own. It's become a genuine force multiplier for the kind of solo technical work I do, and I've leaned into that. But I've never been fully comfortable with it, and I think that discomfort is worth being honest about.</p>

<p>The issue isn't that Claude is adversarial. It isn't. The issue is that the surface area of risk is large and most of it is invisible during normal use. Every file Claude reads, every bash command it runs, every MCP tool it calls is a potential vector. If a project directory contains a malicious string crafted to look like an instruction, Claude might follow it. If a dependency in one of my Node-based MCP servers was compromised, Claude would happily execute it every time I opened that project. If my conversation history — which contains over 300KB of every prompt, code snippet, and environment variable I've ever pasted into a session — ended up readable by the wrong process, that's a significant credential exposure. None of this is theoretical. These are real attack surfaces that exist on any machine running an AI coding assistant with broad file system and shell access.</p>

<p>I've managed this the careful way for a while. Narrow approval modes. Reviewing every tool call before confirming. Not pasting secrets directly into chat. Keeping MCP servers scoped. But managing it manually only works until it doesn't, and the recent public discussion around the Claude Code source code pushed me past the point of being comfortable with a manual-only approach. Not because the leak itself was catastrophic, but because it was a reminder that the gap between "this is probably fine" and "this has been compromised" isn't always visible from where you're sitting.</p>

<p>My laptop does most of the heavy lifting on this setup. It's where I write code with Claude, run initial analysis, and test things before they get offloaded to TrueNAS. That makes it both the most capable node in my homelab and the most exposed one — the first explorer and the canary in the coal mine simultaneously. Something watching it locally, without depending on Claude Code being active, was probably overdue.</p>

<h2 class="section-heading">What I Built</h2>

<p>host-sentry is a small FastAPI service that runs in WSL under systemd and monitors my workstation on a schedule. It doesn't depend on Claude Code, doesn't call out to any external service by default, and doesn't require anything from me day-to-day. It runs, it watches, and it tells me when something changes that I should look at.</p>

<p>The architecture follows the same pattern as the trading bots I've built — FastAPI app, APScheduler for periodic jobs, SQLite for persistence, a live dashboard served locally. That pattern is familiar enough that the whole thing came together quickly, and familiar enough that I trust it. There's no reason to invent a new shape for something like this.</p>

<p>Five things run on a schedule. An integrity watcher hashes a defined set of critical files — Claude's settings, every project's MCP configuration, my shell rc files, SSH authorized keys, memory files I write to persist things across sessions — and alerts when anything changes unexpectedly. A secret scanner walks my project directories and conversation history looking for high-entropy strings and known credential patterns: Alpaca keys, Discord tokens, Anthropic API keys, GitHub PATs, private key headers. A process watcher snapshots listening sockets and established outbound connections and diffs them against a known-good allowlist. A Claude audit module tails my conversation history looking for tool calls matching patterns I've decided are dangerous: curl piped anywhere, base64 decode, writes to authorized_keys, rm -rf from root. And a YARA scanner runs nightly against Downloads and tmp using the same ruleset I use for memory forensics work.</p>

<p>On the Windows side, a PowerShell companion script runs hourly via Task Scheduler and checks Defender status, listening ports, scheduled tasks, startup items, local admins, and WSL network configuration. It POSTs findings to the WSL service over localhost. The whole picture — both sides of the WSL boundary — lands in one dashboard.</p>

<h2 class="section-heading">The Threat Model It's Actually Solving For</h2>

<p>Before building it I wrote out the specific threats I was trying to detect. That exercise is worth doing because "I want security monitoring" is too vague to build something useful from. You end up with either too much noise or coverage in the wrong places.</p>

<p>The thing I care most about is hook abuse. Claude Code supports configuring shell hooks that fire automatically on tool events — before a file edit, after a bash command, on session start. Currently I have none configured. But <code>~/.claude/settings.json</code> and any project-level <code>.claude/settings.local.json</code> can define them, and an attacker who can write to either of those files gets persistent arbitrary code execution under my user on every subsequent Claude session. The integrity watcher fires immediately on any change to those files, before the next session opens.</p>

<p>The second thing is MCP server compromise. Each project's <code>.mcp.json</code> declares a Node process that Claude spawns. A poisoned <code>npm install</code> in any of my MCP server directories is code execution every time I open that project. The integrity watcher covers the package files. The secret scanner covers the directories. Neither is a substitute for auditing dependencies, but both give me a signal if something changes unexpectedly after a dependency update.</p>

<p>The third is credential sprawl. I found during the initial scan that several <code>.env</code> files across my projects had world-readable permissions — a consequence of WSL's default mount behavior, not intentional, but real. My conversation history is 300KB of text that contains fragments of things I've pasted into chat over months, including values that are now rotated but were valid at the time. The secret scanner doesn't just catch new exposures; it gives me a map of where sensitive material has accumulated so I can make deliberate decisions about rotation and cleanup.</p>

<p>The fourth is memory poisoning. I write persistent notes to a memory directory that Claude reads at the start of every conversation. That's useful for continuity but it means the memory files are part of my attack surface — anyone who can write there can plant instructions that persist across sessions. Having the integrity watcher on that directory is roughly equivalent to watching a CI config: changes there are changes to behavior, and they should be reviewed.</p>

<h2 class="section-heading">What It Looks Like Day to Day</h2>

<p>Most of the time it's invisible. The service runs under systemd, starts at login, and sits quietly in the background. When a scan completes and finds nothing new, nothing happens. When it finds something, a single summary notification appears in the Windows system tray — not one notification per finding, which was an early mistake that resulted in a genuinely impressive balloon storm — and the dashboard reflects the new alerts.</p>

<p>The dashboard lives at <code>http://127.0.0.1:8765/dashboard</code> and is the main interface. It polls every thirty seconds, shows unacknowledged alerts with expandable detail rows, scan status cards with next-run countdowns, and the full recent history. It's self-contained HTML served from the same FastAPI process so there are no CORS issues and no dependency on anything external.</p>

<p>Real-time coverage for the most critical paths comes from inotify. Changes to Claude settings files and MCP configurations fire an alert immediately rather than waiting for the next scheduled integrity check. Everything else runs on intervals: integrity every four hours, process checks every six, Claude audit every four, secrets and YARA nightly.</p>

<p>An AI triage module is built in but off by default. When enabled with an Anthropic API key, it routes medium-and-above alerts through Claude for classification — benign, suspicious, or malicious, with reasoning and a recommended action. It's useful for the cases where the raw alert is technically accurate but the right response isn't obvious. I left it off by default because running every alert through an API call when the service is a local security monitor felt like the wrong default, but the option is there when the noise-to-signal ratio makes it worth it.</p>

<h2 class="section-heading">What the Initial Scan Found</h2>

<p>The honest answer is more than I expected. All of my <code>.env</code> files were mode 777 because WSL mounts the Windows filesystem without metadata support unless you explicitly configure it. One project had its API secret key hardcoded in the MCP configuration file — visible to any process that could read that directory, and technically committed to a local repo. My Claude sessions directory had files at 644. A plugin had <code>git push</code> in its auto-approval list, which means Claude could push to remote without prompting. An account called <code>enduser</code> had its password changed with no audit trail — nine minutes after a Cellebrite UFED crash-loop on the same machine, cause still undetermined. Two orphaned Active Directory SIDs were sitting in local group membership with no corresponding accounts attached to them. The built-in Administrator account was still enabled. All three are textbook persistence footholds, and none of them were visible without specifically looking.</p>

<p>The account was disabled, the SIDs were removed, and account management auditing got turned on so future changes leave a trail. The ambiguity around <code>enduser</code> is what stuck with me — not because it was necessarily malicious, but because I couldn't rule it out. That's exactly the kind of finding host-sentry is built to surface before it becomes a question you're answering after the fact.</p>

<p>None of these were the result of malicious activity. All of them were the result of convenience and accumulated configuration drift — the kind of thing that doesn't register as a problem until you look for it specifically. Running the scan wasn't alarming. It was clarifying. The fixes were straightforward once I knew what I was looking at.</p>

<h2 class="section-heading">Why Local</h2>

<p>The whole thing is local by design. No cloud logging, no external webhook by default, no dependency on any service being reachable. This was a deliberate choice rather than a constraint. A security monitor that phones home is a monitor that has its own attack surface, its own availability dependency, and its own data residency question. The machine I'm trying to protect is the one I'm sitting in front of. The alerts should land there too.</p>

<p>The code is at <a href="https://github.com/dfarid479/host-sentry">github.com/dfarid479/host-sentry</a>.</p>
