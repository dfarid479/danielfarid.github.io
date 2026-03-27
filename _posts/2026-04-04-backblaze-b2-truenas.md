---
layout: post
title: "Adding Off-Site Backup to My TrueNAS Homelab with Backblaze B2"
subtitle: "Simple, cheap, and nobody else can read it"
date: 2026-04-04 10:00:00 -0400
categories: homelab
background: '/img/posts/backblaze.png'
---

<p>My homelab gap analysis from a few months back flagged a few things as critical. No resource limits on containers was one. No off-site backups was another. The exact note I wrote was: "all backups on same machine—fire or theft means total loss." That's not a backup strategy. That's just having two copies of the same thing in the same building.</p>

<p>I finally got around to fixing it. The solution was Backblaze B2, and I want to write about it because the setup was simpler than I expected and the cost is almost nothing for what I'm backing up.</p>

<h2 class="section-heading">What I Actually Need to Back Up</h2>

<p>My TrueNAS box runs ZFS with RAIDZ2, which protects against drive failures. It runs daily snapshots, which protect against accidental deletion within a retention window. What neither of those protect against is the whole machine being gone—fire, theft, or a catastrophic failure that takes out the array entirely.</p>

<p>For off-site backup, I'm not trying to replicate the whole homelab. Most of what runs on that box is reconstructible: container images can be re-pulled, media can be re-acquired, config files are in Git. What I care about is data that's genuinely irreplaceable or painful to recreate: the trading bot's SQLite databases and logs, the Obsidian vault (which also syncs through CouchDB, but I want a third copy), .env files and secrets, and a few other datasets that took real time to set up.</p>

<p>That comes out to a few gigabytes. Not a lot. Which is exactly why Backblaze B2 made sense.</p>

<h2 class="section-heading">Why Backblaze B2</h2>

<p>I looked at the usual options. S3 is the obvious choice if you're already deep in AWS, but for a homelab with a few GB of data, the pricing structure and the complexity of IAM permissions felt like overkill. Wasabi is popular in homelab circles. Cloudflare R2 has no egress fees, which matters if you're pulling data back regularly.</p>

<p>For a backup workload—write frequently, read almost never—B2 made the most sense. It's $6 per terabyte per month for storage. My datasets add up to maybe 15 GB. That's under a dollar a month. The first 10 GB of egress per day is free, which is more than enough for the rare restore scenario. The pricing is simple enough that I didn't need a calculator to know it would be cheap.</p>

<p>Backblaze also has a long track record in the homelab and indie developer community. They've been running cloud storage since 2009, their pricing has been stable, and they're not a division of a larger company with different strategic priorities that might affect the service. For something I want to mostly forget about, that reliability matters more than squeezing out the last cent.</p>

<h2 class="section-heading">The Setup</h2>

<p>TrueNAS Scale has cloud sync built in. Under Data Protection, there's a Cloud Sync Tasks section that handles scheduling, credential management, and the actual sync—powered by rclone under the hood. You don't have to touch rclone directly; TrueNAS wraps it in a UI.</p>

<p>Setting it up took three steps.</p>

<p>First, I created a bucket in Backblaze B2. Private, not public. I also created a scoped application key—restricted to that specific bucket, read and write only—rather than using the master account key. If the key ever leaks, the blast radius is contained to this one bucket.</p>

<p>Second, I added the credentials to TrueNAS under System → Credentials → Backup Credentials. Provider is Backblaze B2, paste in the key ID and application key, click Verify Credential. TrueNAS made a test connection and confirmed it worked before saving.</p>

<p>Third, I created the Cloud Sync Task itself. Direction: Push. Transfer Mode: Sync. Source: the datasets I care about on <code>/mnt/main-storage/</code>. Schedule: daily at 2 AM. And then the important part—I enabled client-side encryption and set a passphrase.</p>

<h2 class="section-heading">Client-Side Encryption</h2>

<p>Backblaze offers server-side encryption. I turned it off and used TrueNAS's built-in client-side encryption instead. The difference matters.</p>

<p>Server-side encryption means Backblaze encrypts your data on their end, with a key they manage. They can decrypt it. Their employees can decrypt it, in principle. Any court order or compelled access that reaches Backblaze reaches your data. For a lot of use cases this is fine—the encryption protects against other customers' data being mixed with yours, not against Backblaze itself.</p>

<p>Client-side encryption means your data is encrypted before it leaves your machine. What arrives at Backblaze is ciphertext. They store ciphertext. If their infrastructure were compromised, the attacker gets ciphertext. Backblaze can't read it. I have .env files in these backups with API keys and private key paths. I have a trading bot database. I'd rather those not be readable by anyone other than me, regardless of what happens on Backblaze's end.</p>

<p>The passphrase lives locally. If I lose it, the backup is unrecoverable—that's the tradeoff. I stored it in my password manager and wrote it down in a sealed envelope in a drawer. Belt and suspenders. The backup is useless without it, so treating it with the same care as the data itself makes sense.</p>

<h2 class="section-heading">Running It</h2>

<p>After saving the task, I ran a dry run first. TrueNAS shows you what would be transferred without actually doing it—good for confirming the source paths are right and the credential works against the destination bucket. The dry run completed cleanly, showing the files it would push.</p>

<p>Then I ran it for real. The first sync transferred the full dataset and took a few minutes. Every subsequent run is incremental—only changed files get pushed. For a daily backup of mostly-static data like a SQLite database and config files, the incremental transfers are small. The scheduled 2 AM job is done before anyone's awake.</p>

<p>I can see the transfer logs in TrueNAS and can verify the files landed in the B2 bucket through the Backblaze web UI. The encrypted files show up with rclone-generated names—just hashed strings, no readable filenames. The directory structure is obfuscated too. From Backblaze's perspective, it's an opaque bucket of encrypted blobs with timestamps.</p>

<h2 class="section-heading">What This Actually Cost Me</h2>

<p>Setup time: maybe 30 minutes, including reading through the TrueNAS documentation on cloud credentials and deciding between the encryption options. This was genuinely one of the simpler infrastructure tasks I've done on the homelab.</p>

<p>Ongoing cost: under a dollar a month at my current data size. If I add more datasets, it scales at $6/TB. For a homelab, I don't realistically see hitting a price point that matters.</p>

<p>Ongoing maintenance: essentially none. The task runs on schedule. TrueNAS logs the results. Uptime Kuma monitors the TrueNAS box. If a sync fails, I'll find out. Other than periodically checking that the job is still completing and the bucket is growing as expected, there's nothing to manage.</p>

<h2 class="section-heading">Recommended</h2>

<p>If you're running TrueNAS and haven't set up off-site backup, this is the path of least resistance. The TrueNAS cloud sync integration is genuinely well done—rclone handles the heavy lifting, the UI abstracts the complexity, and the credential verification step catches config errors before you schedule a task and walk away. Backblaze B2 is cheap enough that cost isn't a real consideration at homelab scale. And client-side encryption means you can put sensitive data in the backup without thinking twice about who else might have access to it.</p>

<p>It took one afternoon. I probably should have done it a while ago.</p>
