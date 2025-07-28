---
layout: post
title:  "Teaching Security"
subtitle: "Hands-On WiFi, RFID, and NFC"
date:   2025-07-28 12:01:43 -0400
categories: certs
background: '/img/posts/Wifi_protect_750x500.jpg'
---

<p>Last week I had the opportunity to run a workshop for local law enforcement on wireless security fundamentals. Instead of death-by-PowerPoint, we went hands-on with some affordable hardware that demonstrates real attack vectors they'll encounter in the field.</p>

<h2 class="section-heading">The Setup-Three pieces of kit did all the heavy lifting:</h2>

<p>M5Stack CardPuter with EvilM5Project firmware - This pocket-sized device runs captive portal attacks and demonstrates how easily WiFi networks can be spoofed. The CardPuter's form factor makes it perfect for showing how small and inconspicuous these tools can be. Shoutout to 7h30th3r0n3 for developing this comprehensive penetration testing toolkit.</p>

<p>M5Stack with Bruce firmware - Same hardware platform, different firmware focus. Bruce excels at RFID/NFC operations, letting us clone cards and demonstrate proximity-based vulnerabilities. Thanks to pr3y for creating this versatile RFID/NFC toolkit.</p>

<p>Pwnagotchi on Raspberry Pi Zero 2W - The AI-powered WiFi auditing tool that passively collects WPA handshakes. Its "pet" interface keeps things engaging while demonstrating serious wireless reconnaissance capabilities. Credit to jayofelony for maintaining the current iteration of this project.</p>

<h2 class="section-heading">Single Board Computers: The Perfect Learning Platform</h2>

<p>What makes these single-board computers valuable for training environments is their ability to democratize access to complex cybersecurity concepts. Traditional enterprise-grade penetration testing equipment can cost thousands of dollars and requires specialized knowledge just to configure. These platforms, however, provide a low-barrier entry point where officers can experiment, make mistakes, and learn iteratively without significant financial risk. The M5Stack and Raspberry Pi ecosystems offer extensive documentation, active communities, and modular approaches that allow learners to progress from basic wireless concepts to advanced attack methodologies at their own pace. This hands-on accessibility transforms abstract cybersecurity theories into tangible, understandable demonstrations - making complex RF protocols, encryption mechanisms, and network vulnerabilities immediately comprehensible to practitioners who need to recognize and investigate these threats in real-world scenarios. Perhaps the best part is they can be re-purposed, for example, I have plans to repurpose the Raspberry Pi as a personal VPN, Honeypot, or Internet in a Box.<p>

<h2 class="section-heading">Captive Portal Reality Check</h2>

<p>We started with the EvilM5Project firmware on the CardPuter. Within minutes, officers were creating fake WiFi networks that looked identical to legitimate hotspots. The captive portal functionality showed how easily credentials can be harvested when users connect to malicious access points. I have illustrated this before with M5Stack and the Nemo firmware, but this firmware incorporates many aspects of Nemo with a lot of added features.</p>

<p>The key point: Users will almost always enter their credentials when prompted, especially if the fake network mimics a familiar brand or location. Coffee shops, airports, hotels - all prime targets for this type of attack.

<h2 class="section-heading">RFID/NFC Vulnerabilities</h2>

<p>Switching to Bruce firmware, we demonstrated how many access cards and proximity tokens can be read and cloned. The M5Stack's built-in display made it easy to show real-time card data as we scanned everything from hotel key cards to employee badges.</p>

<p>Most officers were surprised by two things: how quickly cards could be cloned (seconds), and how many cards in their own wallets were readable. I even demonstrated using a student's employer-issued fleet card. The range limitations became apparent too - you need to be within a few centimeters, which has implications for how these attacks happen in practice.</p>

<h2 class="section-heading">WiFi Handshake Collection</h2>

<p>The Pwnagotchi demonstrated passive WiFi monitoring. As it collected handshakes from nearby networks, officers could see how much wireless traffic was constantly available for analysis. The device's AI learns to optimize its handshake collection over time, making it increasingly effective. I illustrated this by using my cellphone hotspot to upload the handshakes in real-time for a rainbow or dictionary attack.</p>

<p>This drove home the point about wireless security: your WiFi traffic is always visible to anyone with the right tools and knowledge. WPA2/WPA3 encryption helps, but handshakes can still be captured for offline analysis.</p>

<h2 class="section-heading">Practical Takeaways</h2>

<p>The hands-on approach revealed several key points:</p>

<ul>
    <li><strong>Physical proximity matterst</strong> - Most of these attacks require the bad actor to be relatively close to targets. This creates opportunities for detection and interdiction.</li>
    <li><strong>Social engineering is the real threat</strong> - Technical vulnerabilities exist, but human behavior amplifies the risk. Users connecting to suspicious networks or ignoring security warnings create the biggest exposure.</li>
    <li><strong>Detection is possible</strong> - Unusual wireless activity can be monitored and flagged. Understanding how these tools work helps in developing countermeasures and investigation techniques.</li>
    <li><strong>Equipment is accessible</strong> - All the hardware we used costs under $200 total and is readily available. This isn't nation-state level stuff - it's script kiddie accessible.</li>
</ul>

<h2 class="section-heading">Why This Matters</h2>

<p>Digital forensics isn't just about analyzing seized devices anymore. Understanding how wireless attacks work, what evidence they leave behind, and how to detect them in progress is becoming essential for modern law enforcement. As cloud capabilities and software-defined networking increase in utilization, these concepts will become hard to ignore in a few years.</p>

<p>The officers left with a better understanding of wireless threat vectors and practical knowledge about what to look for during investigations. More importantly, they got hands-on experience with the actual tools being used in the wild.</p>
