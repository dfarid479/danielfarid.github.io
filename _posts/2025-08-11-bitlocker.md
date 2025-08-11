---
layout: post
title:  "Cracking BitLocker"
subtitle: "When the Front Door is Locked but the Windows are Open"
date:   2025-08-10 15:01:43 -0400
categories: certs
background: '/img/posts/CrackingBitLocker.png'
---

<p>Recently encountered a BitLocker-encrypted Windows 11 Home 23H2 laptop during a digital forensics examination. Here's the methodology I used to gain access and decrypt the system for imaging.</p>

<h2 class="section-heading">The Setup</h2>

<p>Standard drill first - photographed everything, pulled the back cover, found a Western Digital PC SN530 256GB SSD sitting there. Connected it to my Tableau T7u bridge to see what we're dealing with. BitLocker encryption, as expected. No surprises there.
Popped the drive back in and fired up the machine. One user account: "Walt".</p>

<h2 class="section-heading">The Real Work</h2>

<p>Attempted basic password variations first. Rather than continue brute force attempts, I moved to a more systematic approach.Rebooted into BIOS and killed Secure Boot. This is key - you need this disabled to boot from external media on most modern systems.Plugged in my forensic response USB and booted from it. Time for some SAM file surgery. I used NTPWEdit to crack open /system32/config/SAM and reset "Walt's" password. Changed User 1001's password to "temp123". Nothing fancy, just needed something that would work. Something to note here, Windows reserves SID numbers below 1000 for built-in system accounts, services, and special groups. Starting user accounts at 1000 ensures there's no conflict with these reserved identifiers. So If "Walt" had SID 1001, it suggests either:</p>
<ul>
    <li>There was another user account created first (possibly deleted later)</li>
    <li>The system was set up in a domain environment initially, then moved to workgroup</li>
    <li>There was some other account creation that happened before "Walt's" account</li>
</ul>
<p>This is actually good forensic intelligence - it might indicate there were other user accounts on this system that could be worth investigating, even if they're no longer visible in the current user interface.</p>

<h2 class="section-heading">Getting In</h2>

<p>I restarted the machine, logged in as "Walt" with my new password. First stop: disable BitLocker encryption. Once that was done, powered down and reconnected to the Tableau bridge for proper imaging. This approach succeeded because BitLocker's protection relies on the integrity of the local user authentication. By modifying the SAM file offline, we bypass the encryption layer's dependence on user credentials. This technique is effective on local accounts but would not work on domain-joined systems with centralized key management.</p>

<h2 class="section-heading">Alternative Approaches Considered</h2>

<p>Several other methods were available but ultimately less suitable:</p>
<ul>
    <li>Kon-Boot for authentication bypass (less reliable on newer Windows versions)</li>
    <li>Registry analysis for cached tokens (low success probability)</li>
    <li>Cold boot attacks (timing-dependent and unreliable)</li>
</ul>

<h2 class="section-heading">Why This Method Was Optimal</h2>

<p>The SAM file modification approach was selected because it provides reliable access without hardware damage, works consistently on local Windows accounts, requires only standard forensic tools, maintains system integrity for subsequent analysis, and has minimal risk of data corruption. Several Local account password resets remain viable for BitLocker bypass on non-domain systems.</p>

<p>Note: This methodology requires proper legal authorization and should only be applied to systems within scope of legitimate forensic examination.</p>