---
layout: post
title: "Replacing iCloud Sync for Obsidian with a Self-Hosted CouchDB"
subtitle: "How I Got My Notes Off Apple's Servers and Onto My Own Homelab"
date: 2026-03-28 10:00:00 -0400
categories: homelab productivity
background: '/img/posts/syncthing.png'
---

<p>I've been using Obsidian since the homelab idea started. It's the tool that finally clicked for me—local Markdown files, no proprietary format, a linking model that actually maps to how I think. The plugin ecosystem is deep enough to do serious work without feeling like you're assembling a Rube Goldberg machine.</p>

<p>But sync was always the weak point.</p>

<p>I'm running a TrueNAS homelab with twenty-plus containerized services. I have Tailscale, Nginx Proxy Manager, a wildcard SSL cert, and a ZFS array for storage. The one thing I couldn't figure out was how to get my Obsidian vault to sync from my workstation to my iPhone without routing it through Apple's servers.</p>

<p>This is the story of how I finally fixed that.</p>

<h2 class="section-heading">The iCloud Problem</h2>

<p>Obsidian on iOS uses iCloud Drive for sync when you're not paying for Obsidian Sync. In theory, straightforward. In practice, it was never quite right. Sync would stall. Files would show up out of order. Opening the app on my phone sometimes showed vault contents from hours ago even after walking into the same room as my workstation. The reliability just wasn't there.</p>

<p>Beyond the reliability issues, I don't love the idea of my notes—a combination of DFIR research notes, project documentation, and personal knowledge base—sitting in Apple's cloud. Not because I think Apple is reading them, but because data sovereignty matters to me as a principle. I run a homelab for exactly this reason: I want control over where my data lives and who can access it.</p>

<p>Obsidian Sync exists and it probably works great. But I'm already running storage infrastructure. Paying monthly to sync a folder of Markdown files feels like buying groceries when you have a full pantry.</p>

<h2 class="section-heading">What I Already Had</h2>

<p>I'd already solved the Windows-to-TrueNAS side of the problem with Syncthing. The vault files sync continuously between my workstation and a dataset on TrueNAS over Tailscale. Block-level delta sync, works on LAN and remotely, near-instant propagation when files change. I'm not replacing Syncthing—it handles the backup angle well, and having a copy of the vault on TrueNAS matters to me.</p>

<p>What Syncthing can't do is iOS. It doesn't have an iPhone client, and even if it did, the iOS sandbox model would make it difficult to have Syncthing write directly into the Obsidian app container. iOS is a different problem from desktop sync.</p>

<p>I tried Möbius Sync, which is a Syncthing client for iOS. It worked, but Obsidian on iOS expects to own its vault directory inside the iCloud container, and Möbius couldn't write to that path. A dead end.</p>

<p>What I needed was something that worked at the application layer—a sync backend that Obsidian's iOS app could talk to directly. That's a different category of solution.</p>

<h2 class="section-heading">The LiveSync Discovery</h2>

<p>After ruling out filesystem-level approaches, I started looking at Obsidian's community plugin ecosystem. That's where I found <a href="https://github.com/vrtmrz/obsidian-livesync">Self-hosted LiveSync</a> by vrtmrz. It's a community plugin that syncs Obsidian vaults through a CouchDB backend. The plugin handles conflict resolution, chunked storage for large files, and end-to-end encryption. The backend is just CouchDB—a well-documented, Apache-licensed document database that's been in production use since 2005.</p>

<p>CouchDB's replication protocol is what makes this work. It's designed for disconnected, eventually-consistent sync—exactly the problem Obsidian sync needs to solve. The LiveSync plugin wraps it in an Obsidian-native interface with a setup wizard and a config doctor that validates the backend configuration. The whole thing is well thought out.</p>

<p>The catch was that iOS Obsidian requires an HTTPS endpoint. The app refuses to connect to plain HTTP backends. That ruled out direct local network access—I'd need a proper SSL certificate on whatever URL I pointed the plugin at. Fortunately, I already had <code>*.waltbyte.net</code> wildcard certificate in Nginx Proxy Manager and a bunch of services running behind it. Adding one more subdomain was straightforward.</p>

<h2 class="section-heading">The Deployment</h2>

<p>CouchDB runs in a Docker container. I added it to my existing Portainer stack on TrueNAS alongside everything else. The compose block is minimal—the image, a few environment variables for the admin credentials, and two bind mounts: one for the data directory and one for a <code>local.d</code> configuration directory.</p>

<p>CouchDB reads all <code>.ini</code> files in <code>local.d</code> and merges them into its configuration. I created a <code>local.ini</code> there with the settings LiveSync requires: single-node mode, CORS enabled, maximum document and request sizes raised to handle vault chunks, authentication required on all endpoints.</p>

<p>The CORS configuration needed explicit origins for all three Obsidian platforms. Desktop Obsidian uses <code>app://obsidian.md</code>. iOS uses <code>capacitor://localhost</code>. Android uses <code>http://localhost</code>. All three need to be in the <code>origins</code> list or certain clients will get rejected at the preflight stage.</p>

<p>Before starting the container, the data directory needs to be owned by uid 5984—that's the user CouchDB runs as inside the container. If the directory is owned by root, the container starts but can't write to it. I pre-created the TrueNAS datasets and chowned them before deploying the stack. This is the kind of thing that's easy to miss and produces logs that look like permissions errors but don't immediately tell you which layer is wrong.</p>

<h2 class="section-heading">The Init Script</h2>

<p>A fresh CouchDB instance needs a one-time initialization step: <code>_cluster_setup</code> API call to enable single-node mode, then CORS configuration, database creation, and a few other settings. The LiveSync documentation ships a shell script that handles all of it—ten API calls, each checking for success before proceeding.</p>

<p>I ran the script once against the container over the local network. It completed cleanly and returned <code>{"ok": true}</code> on each step. After that I didn't need it again—the config persisted in the CouchDB data directory, and the container has been running correctly through restarts since.</p>

<p>One thing that tripped me up: CouchDB writes the admin password hash to a config file (<code>docker.ini</code>) once you set it. If you ever change the password through the Portainer environment variables after first run, the new value in the env var doesn't match what's in the config file and authentication breaks. Set the password once and leave it. If you need to change it, go through the CouchDB admin UI, not the container environment.</p>

<h2 class="section-heading">SSL and Proxy Setup</h2>

<p>In Nginx Proxy Manager I added <code>sync.waltbyte.net</code> as a new proxy host pointing to the CouchDB container on port 5984. I used the existing wildcard certificate rather than generating a new Let's Encrypt cert for the subdomain—the wildcard already covered it, so no additional certificate management needed.</p>

<p>One thing to verify in the NPM configuration: WebSocket support needs to be enabled for the proxy host. CouchDB's <code>_changes</code> feed uses long-polling in some configurations, and LiveSync's real-time sync benefits from it. I enabled it in the advanced settings tab.</p>

<p>With that in place, I could reach CouchDB at <code>https://sync.waltbyte.net</code> from anywhere—local network, Tailscale, and from my iPhone on cellular. The SSL certificate validated correctly on iOS.</p>

<h2 class="section-heading">LiveSync Setup: Windows</h2>

<p>The LiveSync plugin setup wizard walks you through the connection. You point it at the CouchDB URL, enter the admin credentials, name the database, and it creates everything it needs. The key decision point is end-to-end encryption.</p>

<p>LiveSync's E2EE uses a separate passphrase from your CouchDB credentials. The passphrase encrypts vault content before it ever reaches CouchDB—the database stores ciphertext, not your notes. This matters because CouchDB's admin account could theoretically be compromised at the proxy layer, and I wanted the vault contents protected even in that scenario. The passphrase never leaves your devices; CouchDB never sees the plaintext.</p>

<p>I set a strong E2EE passphrase, completed the wizard, and let it do the initial upload. The first sync takes a few minutes as it chunks and uploads the entire vault. After that, only changed documents replicate.</p>

<h2 class="section-heading">LiveSync Setup: iOS</h2>

<p>On the iPhone, it's the same plugin, same wizard. The first screen asks whether this is a new setup or adding a device to an existing sync. I chose the latter, pasted in the server URL and credentials, entered the E2EE passphrase—which has to match exactly what I set on the desktop—and it connected.</p>

<p>The iOS wizard has one more confirmation step than the desktop: it asks you to verify you understand it's going to overwrite local data with the remote vault. Read this carefully. It means the CouchDB copy becomes authoritative. Since I'd just uploaded from the desktop, that was correct—I wanted the desktop version to win. Confirmed, and it pulled the vault down cleanly.</p>

<p>After setup I disabled iCloud Drive for Obsidian on the iPhone. There's no reason to have it enabled now that sync is handled through CouchDB. The files live in Obsidian's local container on the phone and stay current through LiveSync.</p>

<h2 class="section-heading">How It All Fits Together</h2>

<p>After the full deployment, my sync architecture looks like this:</p>

<ul>
<li><strong>Syncthing</strong>: Windows workstation ↔ TrueNAS, file-level block sync over Tailscale. The TrueNAS copy is a filesystem backup of the vault. This keeps running and doesn't interact with LiveSync.</li>
<li><strong>CouchDB + LiveSync</strong>: Windows workstation ↔ iPhone, real-time document sync over HTTPS. Changes propagate in seconds when both devices are online.</li>
<li><strong>iCloud</strong>: disabled entirely for Obsidian on iPhone.</li>
</ul>

<p>The two sync mechanisms coexist without conflict because they operate at different layers. LiveSync manages document state inside Obsidian. Syncthing manages the raw files on disk on the platforms where it runs. They don't step on each other.</p>

<p>Edit a note on my phone. Open my workstation five minutes later. The change is there. Same in reverse. It's exactly what iCloud was supposed to do but didn't.</p>

<h2 class="section-heading">A Note on Data Sovereignty</h2>

<p>I want to be honest about what this buys you and what it doesn't.</p>

<p>Running CouchDB on your own hardware means your notes aren't on Apple's servers or Obsidian's servers. Your CouchDB admin credentials are in Nginx Proxy Manager's config and your Docker compose file. Your E2EE passphrase lives in the LiveSync plugin config on each device. If either of those is compromised, your notes could be exposed. The security model here is "I trust my homelab infrastructure more than I trust a third-party cloud service"—which may or may not be true for your setup.</p>

<p>For me, with Tailscale for VPN access, Nginx terminating SSL with a proper wildcard cert, and the vault contents encrypted at rest in CouchDB, I'm comfortable with the tradeoff. The data lives on hardware I own, in a building I'm in every day, behind infrastructure I've audited and understand.</p>

<h2 class="section-heading">Was It Worth It?</h2>

<p>Setup time was probably three hours end to end, including the CouchDB container configuration, the Nginx Proxy Manager proxy host, running the init script, and going through the LiveSync wizard on both devices. Not a one-afternoon quickstart, but not a multi-day project either.</p>

<p>The result is sync that actually works. My notes are available on both devices, stay current, and don't pass through any third-party cloud. The end-to-end encryption means the CouchDB database only ever stores ciphertext—the actual vault content is opaque to the infrastructure layer.</p>

<p>If you're already running a homelab with Docker, a reverse proxy, and a wildcard SSL cert, CouchDB plus LiveSync is probably the cleanest self-hosted Obsidian sync solution out there. The LiveSync plugin is actively maintained, the documentation is thorough, and the init script takes care of the tedious CouchDB API calls so you don't have to.</p>

<p>iCloud wasn't working for me anyway. This works.</p>
