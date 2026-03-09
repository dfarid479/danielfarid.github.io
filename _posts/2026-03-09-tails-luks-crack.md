---
layout: post
title: "Cracking Tails OS Persistent Storage"
subtitle: "When the Math Simply Wins"
date: 2026-03-09 09:00:00 -0400
categories: dfir forensics
background: '/img/posts/tails.png'
---

<p>Seized a SanDisk USB during an investigation. Subject declined to provide the encryption password. Standard situation — so we work the problem from the other end. This post covers how I identified the persistent storage partition, imaged the device, and conducted an exhaustive password attack against the LUKS2 encryption. Spoiler: the encryption won. Here's why that's actually the expected outcome, and why the methodology used was the right call regardless.</p>

<h2 class="section-heading">The Device</h2>

<p>The USB in question was a SanDisk, 114.6 GiB total capacity, running Tails OS. For anyone unfamiliar, Tails is a privacy-focused live operating system that boots entirely from USB and leaves no trace on the host machine. The relevant forensic detail is that Tails supports an optional Persistent Storage feature — an encrypted partition on the same USB that survives reboots and stores user-configured data: documents, browser history, credentials, application data, and more. This is the partition that matters. Everything else on the drive is the read-only Tails system image.</p>

<p>Before touching anything, the device was photographed, bagged, logged, and handled according to standard chain of custody procedures. All work was performed on a forensic copy.</p>

<h2 class="section-heading">Imaging</h2>

<p>The USB was imaged using a standard forensic acquisition to produce a raw sector-by-sector image: <code>SanDisk.001</code>. This preserves every byte of the original, including partition tables, slack space, and the full LUKS header — which becomes important later. Working from the image rather than the live device protects against accidental writes and ensures the original evidence is never modified.</p>

<p>To work with the image in Linux, I mounted it as a loop device:</p>

<pre><code>sudo losetup -fP /home/tpkali/SanDisk.001
sudo losetup -l
</code></pre>

<p>This maps the image to <code>/dev/loop1</code> and automatically creates sub-devices for each partition (<code>/dev/loop1p1</code>, <code>/dev/loop1p2</code>).</p>

<h2 class="section-heading">Identifying the Persistent Storage</h2>

<p>With the loop device attached, I ran <code>fdisk</code> to examine the partition layout:</p>

<pre><code>sudo fdisk -l /home/tpkali/SanDisk.001
</code></pre>

<p>Output revealed a GPT-formatted disk with two partitions:</p>

<pre><code>Device                    Start       End   Sectors  Size Type
/home/tpkali/SanDisk.001p1     2048  16775390  16773343     8G EFI System
/home/tpkali/SanDisk.001p2 16777216 240326655 223549440 106.6G Linux reserved
</code></pre>

<p>The layout is exactly what you'd expect from a Tails USB. Partition 1 is the 8GB EFI system partition — the Tails live environment. Partition 2 is the 106.6GB "Linux reserved" partition — that's the Persistent Storage volume. The type designation "Linux reserved" is Tails' way of marking the persistence partition in the GPT table. It's not labeled or advertised, but if you know Tails, you know what you're looking at.</p>

<p>To confirm it was LUKS-encrypted and gather header information:</p>

<pre><code>sudo cryptsetup isLuks -v /dev/loop1p2
sudo cryptsetup luksDump /dev/loop1p2
</code></pre>

<p>The first command confirmed the LUKS signature. The dump revealed the specifics:</p>

<pre><code>Version:       2
Cipher:        aes-xts-plain64
Cipher key:    512 bits
PBKDF:         argon2id
Time cost:     4
Memory:        1048576
Threads:       4
</code></pre>

<p>LUKS version 2. AES-XTS-PLAIN64 with a 512-bit key. And the key derivation function: Argon2id with one gigabyte of memory, time cost 4, and 4 threads. That last part is where the real challenge lives, which I'll get to shortly.</p>

<h2 class="section-heading">Building the Attack</h2>

<p>With no password from the subject and no other avenue for key recovery, the only path forward is a targeted password attack. Since brute-force against modern encryption is computationally unrealistic, the approach is intelligence-driven: build a wordlist from everything known about the subject, then run permutations against it.</p>

<p>OSINT collection on the subject produced a set of candidate terms — names, pets, usernames, years, known patterns. These formed the initial list: 39 candidates covering the most likely base passwords. From there, I generated a permutation list applying common password construction patterns: appended numbers, special character suffixes, case variations, combined terms, and common "complexity" modifiers people use to satisfy password requirements. That expanded the attack surface to 14,491 total candidates.</p>

<p>The attack used a bash loop against the live image via cryptsetup's <code>--test-passphrase</code> flag — no write operations, no decryption, just a direct test of each candidate against the LUKS2 keyslot:</p>

<pre><code>while IFS= read -r password; do
    echo "Trying: $password"
    if echo "$password" | sudo cryptsetup luksOpen --test-passphrase /dev/loop1p2 2>/dev/null; then
        echo "PASSWORD FOUND: $password"
        break
    fi
done &lt; "PasswordList.txt"
</code></pre>

<p>The session ran from the afternoon of March 5th through the morning of March 6th — approximately 17.5 hours. Every candidate was tested. None matched.</p>

<h2 class="section-heading">Why This Approach Was Correct</h2>

<p>Two main alternatives exist for attacking LUKS2: dictionary/rule-based attacks via hashcat (GPU-accelerated), and direct keyslot testing via cryptsetup (what was done here). The choice depends heavily on the PBKDF in use.</p>

<p>For LUKS2 volumes protected by PBKDF2 or bcrypt, hashcat (mode 29200) provides substantial GPU acceleration. You extract the hash header with <code>hashcat-utils/luks2hashcat</code>, run it against a GPU cluster, and throughput can reach tens of thousands of candidates per second depending on hardware.</p>

<p>Argon2id changes that calculus entirely. It is intentionally designed to be resistant to GPU and ASIC acceleration through its memory-hard construction. Each hash derivation requires the full memory allocation — in this case, one gigabyte — to be written and read in full. GPUs have fast cores but limited per-thread memory bandwidth. You can't parallelize Argon2id across thousands of GPU cores the way you can with simpler hash functions, because each thread needs exclusive access to its 1GB working set. The result is that GPU acceleration provides marginal speedup over CPU for Argon2id, and in some configurations CPU is actually more efficient.</p>

<p>With the header parameters confirmed (argon2id, time=4, memory=1048576, threads=4), the direct cryptsetup approach on modern hardware was testing roughly one candidate every 2-4 seconds. That's not a tooling limitation — that's the function doing exactly what it was designed to do. A hashcat GPU run against this same header would have yielded comparable per-candidate timing, not the orders-of-magnitude speedup GPU acceleration provides against weaker KDFs.</p>

<p>Given that reality, a targeted high-quality wordlist with intelligent permutations was the correct strategy. Throwing rockyou at it would take weeks or months for no practical gain. Intelligence-driven candidate generation is how you work a real case under realistic time constraints.</p>

<h2 class="section-heading">Why the Encryption Won</h2>

<p>The attack failed, and that outcome isn't surprising once you understand the math. It comes down to two things working together: the strength of the key derivation function and the quality of the password.</p>

<p>Argon2id was the winner of the Password Hashing Competition in 2015 — the outcome of a years-long effort by the cryptographic community to produce a KDF specifically resistant to modern hardware attacks. The parameters used here (1GB memory, time cost 4) are aggressive. They mean that any attacker — regardless of whether they're using a laptop or a GPU cluster — faces the same fundamental bottleneck: time and memory per attempt. There's no algorithmic shortcut. The 512-bit AES key derived from the password cannot be recovered without the password, and the password cannot be recovered from the LUKS header without testing candidates one by one at the rate the KDF imposes.</p>

<p>The second factor is the password itself. The entire premise of a dictionary attack is that people choose passwords from a predictable space: common words, personal identifiers, simple patterns. The 14,491 candidates tested here covered the full OSINT-derived attack surface — every known name, pet, username, date, and personal identifier associated with the subject, combined with the full range of common permutation patterns. That list should have been sufficient to crack a typical human-chosen password. It wasn't.</p>

<p>What that tells you: either the password contains a component with no OSINT anchor (a random string, an unrelated word, a passphrase fragment not derivable from anything in the public record), or the password length and entropy push it beyond the permutation space we could reasonably construct. Either way, the password obeyed the two fundamental rules of good password creation: no personal information, and sufficient length or randomness to resist pattern-based attacks. Combined with Argon2id's hardware-resistant key stretching, the result is a system that simply can't be broken on any realistic investigative timeline.</p>

<h2 class="section-heading">The Bottom Line</h2>

<p>The methodology used here — forensic imaging, LUKS header analysis, targeted OSINT-based wordlist attack, intelligent permutation generation — represents the correct approach given the available information. There was no faster path. GPU acceleration against Argon2id doesn't provide the speedup that makes brute-force feasible against well-chosen passwords, and throwing generic wordlists like rockyou at an Argon2id container is an exercise in burning time with near-zero probability of success.</p>

<p>This is what defense-grade encryption looks like when it's properly used. Tails made the right call implementing LUKS2 with Argon2id as the persistence layer. A subject who chose a strong password and used Tails as intended created a forensic barrier that reflects the current state of the art — not a failure of methodology, but a demonstration that the cryptography is working exactly as designed.</p>

<p>Note: All work was conducted under proper legal authority as part of a lawful digital forensic examination.</p>
