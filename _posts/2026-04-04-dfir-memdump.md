---
layout: post
title: "dfir-memdump"
subtitle: "Turning RAM Captures Into Actionable Intelligence, Locally"
date: 2026-04-04 12:26:00 -0400
categories: dfir forensics tools
background: '/img/posts/ram.png'
---

<p>I capture RAM on almost every scene I work. It's part of the collection checklist at this point — device is live, collection kit comes out, memory goes first. That part is well established. What's less established, at least in my own workflow, is what happens to that image afterward.</p>

<p>The honest answer most of the time: it gets imaged, hashed, bagged, logged, and then sits on a drive while the investigation moves forward on artifacts that are faster to process. Disk forensics has a mature toolchain. Network forensics has a mature toolchain. Memory forensics has Volatility, which is powerful and deeply capable, but the path from "here's a raw image" to "here's what's actually happening in this machine" has a lot of manual steps and a high floor for entry. The result is that RAM — which is often the most volatile and time-sensitive evidence on the whole scene — ends up being the least utilized data point in the investigation.</p>

<p>The motivation for building this came from two directions. The first is the cyber response side. I wanted something that could run locally on an endpoint during an active incident — no sending images off-site, no waiting on a lab queue — and that could also give a fast first read when someone comes in reporting a compromise. The triage report needs to be good enough to tell you whether you're looking at a real incident and what the attacker was doing, in the time it takes to have the first conversation with the affected party.</p>

<p>The second is criminal forensics, specifically cases involving the possession of unlawful files. That work has a growing encryption problem. Whether it's because the barrier to using full-disk encryption has dropped, because suspects are becoming more aware of forensic methods, or simply because encryption is more prevalent across all consumer devices — the result is the same: more devices that are live and mounted at collection time, and more cases where the question isn't whether contraband exists but whether we can access it. Memory capture is one of the few collection techniques that addresses that directly. The key material is in RAM while the system is running. If you don't get it there, you don't get it.</p>

<p>Those two use cases shaped the tool in different ways, but the core requirement was the same: hand it a raw image, walk away for a few minutes, and come back to a prioritized summary of what's worth looking at. No manual plugin chaining, no parsing JSON by hand, no rebuilding the same analysis workflow every case. Just signal.</p>

<h2 class="section-heading">What's Actually in Memory</h2>

<p>It's worth being specific about why RAM matters, because "volatile evidence" gets said so often it can start to feel like a checkbox rather than a reason. Here's what a memory image can contain that disk forensics simply cannot recover after the fact:</p>

<p>Running processes that have no corresponding file on disk. Fileless malware, reflective DLL injection, process hollowing — all of these techniques exist specifically to avoid writing to disk, and all of them are visible in memory. The injected shellcode sitting in a PAGE_EXECUTE_READWRITE region of a legitimate process doesn't exist anywhere on the filesystem. If you don't grab memory before shutdown, it's gone.</p>

<p>Network connections in ESTABLISHED state. The disk tells you what was installed. Memory tells you what was actively communicating at the moment of collection — the live socket table, the foreign addresses, the processes behind each connection. That's the difference between knowing malware was present and knowing it was actively beaconing to infrastructure at the time you arrived.</p>

<p>Decrypted content. Full-disk encryption protects data at rest. A running system has already decrypted everything it needs to operate. Credentials in LSASS, encryption keys in process memory, plaintext values that are ciphertext on disk — all of it is accessible in a live capture in a way it simply isn't from a disk image alone.</p>

<p>The limitation has never been what's in memory. The limitation has been the time cost of getting real meaning from it.</p>

<h2 class="section-heading">Building the Tool</h2>

<p>dfir-memdump is a Python tool that sits on top of Volatility3 and adds an intelligence layer between the raw plugin output and the investigator. The architecture is straightforward: run a fixed set of Volatility3 plugins against the image, pass all the output through a set of analysis modules, and render the results into a report that a human can actually read.</p>

<p>The plugin stack covers the core artifacts: process list and tree, network connections, suspicious VAD regions via malfind, command lines, loaded DLLs, open handles, token privileges, and BitLocker keys. Eight plugins, all run automatically, all output parsed into typed models so the analysis layer has clean structured data to work with.</p>

<p>On top of that sits ten intelligence modules. The ones that tend to produce the most immediately actionable output in practice:</p>

<p><strong>Anomaly Detector</strong> — checks every process for parent-child relationships that don't make sense (Word spawning PowerShell, explorer.exe spawning net.exe), name masquerading (svchost.exe running from the wrong path), and hollow process indicators. These are the behavioral anomalies that show up before any hash or signature matching is possible.</p>

<p><strong>C2 Detector</strong> — matches outbound connections against the Feodo tracker IP blocklist and a local list of known malicious infrastructure. Also flags connections on ports commonly associated with C2 frameworks and tools like Cobalt Strike, Meterpreter, and Sliver.</p>

<p><strong>Mutex Checker</strong> — this one is underappreciated. Named mutexes are how malware prevents multiple instances of itself from running simultaneously, and they are highly family-specific. WannaCry has a mutex. Cobalt Strike beacons have GUID-style global mutexes. LockBit has a mutex. The module checks open handles against a list of known-bad mutex signatures and also flags any process holding a Process-type handle to another process — the classic OpenProcess → WriteProcessMemory → CreateRemoteThread injection setup.</p>

<p><strong>Privilege Checker</strong> — inspired by PrivHound, this module looks at the token privileges held by each non-system process. SeDebugPrivilege on a process that isn't a debugger is a significant red flag. SeImpersonatePrivilege in the wrong place is a Potato attack waiting to happen. When both are present on the same process, the finding escalates to CRITICAL — that combination is generally sufficient for full domain compromise from a low-privilege starting point.</p>

<p><strong>VirusTotal Client</strong> — submits SHA256 hashes of process images to the VirusTotal API and surfaces any detections. Rate-limited, cached locally in SQLite so you're not burning API quota on hashes you've already checked, and skippable entirely with a flag for offline or air-gapped work.</p>

<p><strong>Encryption Key Finder</strong> — this one is driven directly by the criminal forensics side of the work. Encrypted devices are appearing more often, and a live memory capture at the time of seizure is often the only opportunity to recover key material. If a BitLocker-encrypted drive is present and the system is running, the Full Volume Encryption Key is often sitting in memory. Volatility3's windows.bitlocker.Bitlocker plugin can pull it out directly. The module also runs aeskeyfind and bulk_extractor as subprocess tools if they're installed, scanning the raw image for AES key schedules by detecting the statistical properties of expanded key material — this covers BitLocker, VeraCrypt, TrueCrypt, and any other AES-based cipher. For VeraCrypt and TrueCrypt specifically, it checks whether the mounting process was running at collection time; if it was, the master key is typically in its VAD and recoverable with the same tooling. Recovered keys land in a dedicated section of the report with the dislocker and bdemount commands pre-populated. If the plugin runs and finds nothing — because the drive wasn't encrypted, or because the system had already been shut down and restarted — the report says so explicitly rather than leaving a blank section. Chain-of-custody note: key recovery is performed against the forensic copy of the memory image and documented in the report.</p>

<h2 class="section-heading">Running It</h2>

<p>Installation is standard Python — clone, pip install, done. The only external dependency is Volatility3, which needs to be on PATH. A VirusTotal API key is optional but easy to add via a .env file.</p>

<pre><code>dfir-memdump analyze /evidence/RAM.raw</code></pre>

<p>That's the full command for a default run. It produces three report files — JSON, Markdown, and HTML — in a reports directory. The terminal output gives you the executive summary and a severity-sorted findings table while the full reports are being written.</p>

<p>For faster turnaround when you don't need VT lookups:</p>

<pre><code>dfir-memdump analyze /evidence/RAM.raw --no-vt --format html</code></pre>

<p>The HTML report is the most useful artifact for casework. It's fully self-contained — one file, no internet required to view it — and covers everything in a readable format: the image hash for chain of custody, the executive summary, a process risk leaderboard sorted by weighted finding score, the process tree with suspicious processes flagged, a chronological event timeline, the attack chain reconstruction ordered by kill-chain stage, a dedicated encryption key section with mount commands, and an intelligence findings section grouped by severity level so you see the critical items immediately without scrolling through everything. IOCs are summarized by type — N IPs, N hashes, N mutexes — with the full list expandable when you need it. There's a Print / Save PDF button in the header for generating a clean printable version.</p>

<h2 class="section-heading">The Attack Chain</h2>

<p>One of the features I put the most thought into is the attack chain reconstruction. After all the intelligence modules run, the tool groups findings by their MITRE ATT&CK tactic and orders them along the kill chain — Initial Access through Impact — generating a plain-English narrative for each observed stage.</p>

<p>The output looks something like this: Privilege Escalation was observed — SeDebugPrivilege and SeImpersonatePrivilege were both enabled on process X, sufficient for full domain compromise. Command and Control was observed — one connection to a known C2 IP with X detections on VirusTotal. Impact was observed — a mutex matching the LockBit ransomware signature was found in process Y.</p>

<p>That's the kind of summary you can hand to a supervisor or include in an affidavit without requiring them to understand what PAGE_EXECUTE_READWRITE means. It ties the technical findings to a coherent narrative about what the attacker was doing and what stage they had reached at the time of collection. For investigations where the memory image is one artifact among many, that framing is genuinely useful.</p>

<h2 class="section-heading">Where It Fits</h2>

<p>This isn't a replacement for a full Volatility workflow. There are things the tool doesn't do — no registry hive extraction, no shimcache parsing, no deep artifact carving — and for complex cases those manual deep-dives are still necessary. The purpose is different: get through the first pass quickly, surface what's worth looking at, and let the investigator make informed decisions about where to spend time.</p>

<p>On the cyber response side, that means having an answer ready before the initial triage conversation is over — yes this is a real incident, here's what was running, here's where it was calling out to. On the criminal forensics side, it means leaving every scene knowing whether the memory image contains key material that will matter later, rather than finding out months down the line when the image has been sitting on a shelf.</p>

<p>The memory image has always been worth collecting. The gap has been in making it worth reading before the investigation has moved on. That's what this is trying to close.</p>

<p>The code is open source at <a href="https://github.com/dfarid479/dfir-memdump">github.com/dfarid479/dfir-memdump</a>.</p>
