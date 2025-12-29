---
layout: post
title: "Homelab Chronicles"
subtitle: "Updates, Failures, and Getting My SIEM On"
date: 2025-12-29 15:27:43 -0400
categories: homelab
background: '/img/posts/homelab2.jpg'
---

<p>So I built a homelab three months ago—a proper production-grade personal cloud infrastructure running 17+ services. Since then, I've learned that "production-ready" doesn't mean "maintenance-free." I figured a little update could be helpful.</p>

<p>For the most part my services have been running flawlessly. No issues with my mobile device integration,; it's been great. In this update, a routine TrueNAS update that broke DNS resolution across my entire network, a three-day battle with Wazuh SIEM that ended in defeat, and how a 15-minute Graylog deployment finally gave me the security monitoring I actually needed.</p>

<h2 class="section-heading">The Update That Broke Everything</h2>

<p>TrueNAS had been nagging me about updates for a couple weeks. I figured "how bad could it be?" The system had been rock-solid for months and every piece of data I own is on this thing so I felt like updates shouldnt be avoided. The update went fine—system rebooted, came back up, all services started. Then I tried to access anything. Nothing worked.</p>

<p>The problem was DNS resolution. Tailscale IP addresses had shifted during the update, and my Cloudflare DNS records were still pointing to the old IPs suddenly resolved to nowhere. Every subdomain I'd configured was broken.</p>

<p>What should have been a 10-minute fix turned into an evening of troubleshooting because I hadn't documented which services relied on which IP configurations. The fix was straightforward—update DNS records in Cloudflare, restart Pi-hole. But the real lesson was documentation: I spent the next evening properly mapping my entire network topology, IP allocations, and service dependencies.</p>

<h2 class="section-heading">The Wazuh SIEM Disaster</h2>

<p>With the network stable, I decided to implement proper security monitoring. After some research I settle on Wazuh—an enterprise-grade open-source SIEM that looked perfect on paper. Enterprise features, agent-based monitoring, threat detection, MITRE ATT&CK integration.</p>

<p>"I'll just deploy the Docker stack," I thought. "It'll be running in 10 minutes."</p>

<p>Three hours later, I was staring at SSL certificate errors. The Wazuh indexer refused to start. Certificate validation failures everywhere. I tried disabling SSL requirements with every environment variable I could find: DISABLE_INSTALL_DEMO_CONFIG=true, plugins.security.disabled=true, OPENSEARCH_SSL_VERIFICATIONMODE=none.</p>

<p>Guess WHAT: You cannot disable SSL in Wazuh. It's baked into every component at a fundamental level.</p>

<p>Attempt two: Use Wazuh 4.3.10—older, more stable. Same SSL certificate errors. Different version, identical problems. The indexer crashed: "access denied reading /etc/wazuh-indexer/certs/indexer.pem". Fixed permissions. It crashed again: "certificate is valid for demo.indexer, not wazuh-indexer".</p>

<p>Attempt three: Follow Wazuh's official certificate generation process. Downloaded their cert generation tool, created proper certificates with correct hostnames:</p>

<pre><code>nodes:
  indexer:
    - name: wazuh-indexer
      ip: 172.20.0.26
  server:
    - name: wazuh
      ip: 172.20.0.25
  dashboard:
    - name: wazuh-dashboard
      ip: 172.20.0.27
</code></pre>

<p>Generated certificates. Mounted them correctly. The indexer started for 30 seconds, then crashed with "bad_certificate" errors—despite having certificates it just generated using Wazuh's own tool.</p>

<p>After 8+ hours across three attempts, I gave up. I'd learned a lot about OpenSearch security and SSL/TLS (which I'm sure will be super useful- never), but I didn't have a working SIEM.</p>

<h2 class="section-heading">Graylog: 15 Minutes to Working SIEM</h2>

<p>Time for a reality check. If I've learned anything from my Incident Response training I could accomplish this with LOGS. I needed to see Docker logs in one place, catch failed SSH attempts, and have searchable logs. I didn't need enterprise-grade security orchestration or MITRE ATT&CK integration. I figured with the looming role of being the official SysAdmin at work, I might as well get knee deep know.</p>

<p>Graylog deployment: 15 minutes from docker-compose to working web UI. Three containers (MongoDB, OpenSearch, Graylog), no SSL certificates, no security plugin battles. Just services that start and work:</p>

<pre><code>services:
  mongodb:
    image: mongo:6.0
  opensearch:
    image: opensearchproject/opensearch:2.11.0
    environment:
      - "plugins.security.disabled=true"
  graylog:
    image: graylog/graylog:5.2
    ports:
      - "9001:9000"      # Web interface
      - "1514:1514/udp"  # Syslog
      - "12201:12201/udp" # GELF
</code></pre>

<p>Getting logs flowing was trivial. For TrueNAS: created a Syslog UDP input in Graylog, pointed TrueNAS at it. Done. For Docker containers, updated /etc/docker/daemon.json:</p>

<pre><code>{
  "log-driver": "gelf",
  "log-opts": {
    "gelf-address": "udp://10.0.0.96:12201",
    "tag": "{{.Name}}"
  }
}
</code></pre>

<p>For existing containers in Portainer stacks, added logging config to each service:</p>

<pre><code>logging:
  driver: gelf
  options:
    gelf-address: "udp://10.0.0.96:12201"
    tag: "{{.Name}}"
</code></pre>

<p>Redeployed stacks. Logs started flowing immediately from all 17+ containers.</p>

<h2 class="section-heading">Building Dashboards That Actually Help</h2>

<p>Created a "Homelab Overview" dashboard with widgets that matter:</p>

<p>**Top Active Containers** - Shows which services are chattiest (Immich logs everything).</p>

<p>**SSH Failed Logins** - Search: message:"Failed password". Shows bot brute-force attempts.</p>

<p>**Error Timeline** - Search: level:< 4 (emergency, alert, critical, error). Red area chart showing problems across all services.</p>

<p>**Container Restarts** - Catches flapping services. qBittorrent occasionally restarts—something to investigate.</p>

<p>Total time to create useful dashboards: was probably 60 minutes. No fighting the tool, just building what I needed.The UI isn't super straightfoward and I got hung up a bit. Be careful to make sure you know how the Indexer is naming logs from the stream.</p>

<h2 class="section-heading">Other Updates</h2>

<p>**Kali Linux VM:** Deployed on NVMe storage, accessible via web console. Only runs when needed for security research. Isolated environment with full toolkit (Burp Suite, Metasploit, nmap, Wireshark).</p>

<p>**qBittorrent Performance:** Still glacially slow through ProtonVPN. Tried different endpoints, connection tweaks, nothing helps. Downloads that should take hours take days. It works, just slowly—on the "fix when bored" list.</p>

<p>**Everything Else:** Rock solid. Plex streams 4K without stuttering.Which came in clucth to avoid the Christmas movie streaming price gauging this year. Immich processes photos instantly. Vaultwarden syncs passwords perfectly. Tailscale provides VPN access from anywhere. The core homelab just works.</p>

<h2 class="section-heading">Lessons Learned</h2>

<p>Know When to Walk Away: I spent 8+ hours fighting Wazuh. Should have quit after attempt two. Graylog took 15 minutes and was immediately useful. Sometimes "good enough" beats "perfect but doesn't work."</p>

<p>Documentation Saves Lives: When DNS broke, I couldn't remember which services needed which configurations. Now I have network diagrams, IP allocations, and dependency maps. This has already saved hours during troubleshooting.</p>

<p>Match Tools to Requirements: Wazuh has impressive features I don't need. Graylog does exactly what I need without overwhelming complexity. For a 17-container homelab, simple and functional beats feature-rich and broken.</p>

<p>Monitoring Changes Everything:Before Graylog, I knew when services went down but not why. Now I can see patterns, identify root causes in minutes instead of hours, have alerts in place for any malicious behavior, and actually understand what's happening across my infrastructure. </p>

<h2 class="section-heading">Current State</h2>

<p>Homelab is back to stable production. Seventeen containers with 99%+ uptime. Memory at 40GB/64GB, CPU rarely exceeds 30%, storage at 12% utilization. Graylog ingests 50,000 log messages daily. Monitoring dashboards show what matters. Family members use services without realizing anything changed.</p>

<p>Most importantly: infrastructure I completely control with full visibility into everything happening.</p>

<h2 class="section-heading">For Anyone Building Their Own</h2>

<p>Start with Graylog unless you need specific enterprise features. Don't waste days fighting Wazuh's SSL requirements.</p>

<p>Document your network before you need it. Future you will thank present you. Seriously that sucked.</p>

<p>Accept imperfection. My qBittorrent is slow. Some containers restart occasionally. It's fine because the goal is functional, monitored, maintainable infrastructure you enjoy working on and to learn while doing it.</p>

<p>The homelab journey is iterative—you build, it breaks, you fix, you learn, you improve. Three months ago I thought it was "done." Today I know it's never done. And that's what makes it interesting and I'm sure will have a ton of iterations. Questions? Hit me up. Want to see my configs? They're probably in one of my GitHub repos. Want to tell me what I did wrong? You're probably right, and I'd love to hear about it.</p>