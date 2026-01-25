---
layout: post
title: "When Your ISP Says You're Under Attack"
subtitle: "And Why You Probably Aren't"
date: 2026-01-25 09:00:00 -0400
categories: homelab security
background: '/img/posts/xfinity.jpg'
---

<p>So there I was, Saturday morning, coffee in hand, when my phone buzzes with an Xfinity Advanced Security alert. The message was ominous: "Attempt from IP 194.180.49.117 was blocked on Generic Brand Linux. This happens when a known source of hacking tries to attack a device on your home network."</p>

<p>My TrueNAS server was apparently under attack. Cool. Great way to start the weekend.</p>

<h2 class="section-heading">The Initial Panic</h2>

<p>When you're running 17+ containers, and your entire homelab infrastructure on one box, "known source of hacking" gets your attention. My first thought was that something got exposed—maybe a misconfigured Cloudflare tunnel, a rogue UPnP port forward, or some container decided to phone home to the wrong neighborhood.</p>

<p>I did a quick sanity check. Pulled up my Cloudflare DNS records—all pointing to my Tailscale IP (100.64.0.0/10 range), not my actual public IP. Good. Checked the Xfinity gateway for port forwards—none configured. Also good.</p>

<p>Then I ran a quick port scan against my own public IP:</p>

<pre><code>for port in 22 80 443 445 8080 8443 9000 32400 6881; do
  timeout 2 bash -c "echo >/dev/tcp/[PUBLIC_IP]/$port" 2>/dev/null && echo "Port $port: OPEN" || echo "Port $port: closed"
done
</code></pre>

<p>All closed. Nothing exposed. So what gives?</p>

<h2 class="section-heading">The Second Alert</h2>

<p>The next morning, another alert. Different IP this time: 185.156.73.181. Same story—blocked attempt on "Generic Brand Linux."</p>

<p>At this point I figured it was time to actually investigate these IPs and run a proper security assessment of my stack. If someone's knocking, I want to know who and make sure the door is actually locked.</p>

<h2 class="section-heading">Threat Intelligence</h2>

<p>I dug into both IPs to see what we were dealing with.</p>

<p><strong>IP #1: 194.180.49.117</strong> belongs to MEVSPACE sp. z o.o. (AS201814), a Polish VPS hosting provider based in Warsaw. MEVSPACE has been documented in threat intelligence reports as infrastructure commonly abused for session hijacking and credential stuffing campaigns. Their IPs frequently appear in automated scanning operations. Source: <a href="https://ipinfo.io/AS201814">IPinfo.io</a>, <a href="https://cybersecuritynews.com/hackers-abuse-vps-servers/">CyberSecurityNews</a></p>

<p><strong>IP #2: 185.156.73.181</strong> is registered to TOV E-RISHENNYA (AS211736), with the network "Reldas-net." It's registered in the Netherlands but the actual organization address is in Kyiv, Ukraine. The abuse contact? A Gmail address: erishennya.res@gmail.com. That's always a red flag. This is classic bulletproof-adjacent hosting—cheap VPS instances used for scanning campaigns, then burned when they get blacklisted. Source: <a href="https://ipinfo.io/AS211736/185.156.73.0/24">IPinfo.io</a></p>

<h2 class="section-heading">The Full Security Audit</h2>

<p>Since I was already in investigation mode, I ran a comprehensive security posture assessment on my TrueNAS box. Here's the script:</p>

<pre><code>#!/bin/bash
# TrueNAS Security Posture Assessment

echo "=============================================="
echo "  TRUENAS SECURITY POSTURE ASSESSMENT"
echo "  $(date)"
echo "=============================================="

echo -e "\n[1/15] SYSTEM INFORMATION"
echo "Hostname: $(hostname)"
echo "TrueNAS Version: $(cat /etc/version 2>/dev/null)"
echo "Kernel: $(uname -r)"
echo "Uptime: $(uptime -p)"

echo -e "\n[2/15] USER ACCOUNTS & ACCESS"
echo "--- Users with shell access ---"
cat /etc/passwd | grep -v nologin | grep -v false | grep -v /bin/sync

echo -e "\n--- Users with UID 0 (root equivalent) ---"
awk -F: '$3 == 0 {print $1}' /etc/passwd

echo -e "\n[3/15] SSH CONFIGURATION"
SSHD_CONFIG="/etc/ssh/sshd_config"
if [ -f "$SSHD_CONFIG" ]; then
  echo "PermitRootLogin: $(grep -i "^PermitRootLogin" $SSHD_CONFIG 2>/dev/null || echo 'not set')"
  echo "PasswordAuthentication: $(grep -i "^PasswordAuthentication" $SSHD_CONFIG 2>/dev/null || echo 'not set')"
  echo "PubkeyAuthentication: $(grep -i "^PubkeyAuthentication" $SSHD_CONFIG 2>/dev/null || echo 'not set')"
fi

echo -e "\n[4/15] NETWORK LISTENERS"
echo "--- Services on 0.0.0.0 (all interfaces) ---"
ss -tlnp | grep "0.0.0.0" | awk '{print $4, $6}' | sort -u

echo -e "\n[5/15] FIREWALL STATUS"
iptables -L -n 2>/dev/null | head -20

echo -e "\n[6/15] DOCKER SECURITY"
echo "--- Privileged containers ---"
docker ps -q 2>/dev/null | xargs -I {} docker inspect --format '{{.Name}}: Privileged={{.HostConfig.Privileged}}' {} 2>/dev/null | grep "true"

echo -e "\n--- Containers with host network ---"
docker ps -q 2>/dev/null | xargs -I {} docker inspect --format '{{.Name}}: {{.HostConfig.NetworkMode}}' {} 2>/dev/null | grep "host"

echo -e "\n--- Docker socket mounts (container escape risk) ---"
docker ps -q 2>/dev/null | xargs -I {} docker inspect --format '{{.Name}}: {{range .Mounts}}{{.Source}} {{end}}' {} 2>/dev/null | grep "docker.sock"

echo -e "\n[7/15] APPLICATION AUTH STATUS"
echo "--- Checking *arr apps for auth ---"
for app in sonarr radarr prowlarr; do
  config=$(docker exec $app cat /config/config.xml 2>/dev/null)
  if [ -n "$config" ]; then
    auth=$(echo "$config" | grep -oPm1 "(?<=&lt;AuthenticationMethod&gt;)[^&lt;]+" || echo "None")
    echo "$app: $auth"
  fi
done 2>/dev/null

echo -e "\n[8/15] FAILED LOGIN ATTEMPTS"
grep -i "failed\|invalid" /var/log/auth.log 2>/dev/null | tail -10

echo -e "\n[9/15] SMB SECURITY"
testparm -s 2>/dev/null | grep -i "hosts allow"

echo -e "\n[10/15] NFS EXPORTS"
showmount -e localhost 2>/dev/null

echo -e "\n[11/15] FILE PERMISSIONS"
echo "--- World-writable files in /etc ---"
find /etc -type f -perm -002 2>/dev/null | head -5

echo -e "\n[12/15] CRON JOBS"
crontab -l 2>/dev/null | grep -v "^#"

echo -e "\n[13/15] TAILSCALE STATUS"
tailscale status 2>/dev/null || docker exec tailscale tailscale status 2>/dev/null

echo -e "\n[14/15] CERTIFICATE EXPIRY"
echo "--- Certs expiring within 30 days ---"
find /etc -name "*.crt" 2>/dev/null | head -5 | while read cert; do
  expiry=$(openssl x509 -enddate -noout -in "$cert" 2>/dev/null | cut -d= -f2)
  if [ -n "$expiry" ]; then
    exp_epoch=$(date -d "$expiry" +%s 2>/dev/null)
    now_epoch=$(date +%s)
    days_left=$(( (exp_epoch - now_epoch) / 86400 ))
    [ "$days_left" -lt 30 ] && echo "EXPIRING: $cert ($days_left days)"
  fi
done

echo -e "\n[15/15] RECENT CONFIG CHANGES"
find /etc -type f -mtime -7 2>/dev/null | head -10

echo -e "\n=============================================="
echo "  ASSESSMENT COMPLETE"
echo "=============================================="
</code></pre>

<h2 class="section-heading">The Results: Everything Was Fine</h2>

<p>Here's what my audit confirmed: SSH service disabled (inactive). SSH config set to key-only auth with no root password. SMB access restricted to Tailscale IPs only (hosts allow = 100.64.0.0/10). NFS had no exports configured. All *arr apps had Forms authentication enabled. Only Tailscale ran as a privileged container (which is required). Zero failed login attempts. All external ports closed.</p>

<p>My perimeter was solid. The scanners were hitting a brick wall.</p>

<h2 class="section-heading">What's Actually Happening</h2>

<p>Every public IP on the internet gets scanned thousands of times daily. This isn't news—it's just the background radiation of the modern internet. What's changed is the scale and sophistication.</p>

<p>These alerts represent automated reconnaissance from cheap VPS instances. The attackers spin up servers on MEVSPACE, E-RISHENNYA, and dozens of similar providers, run internet-wide scans looking for exposed NAS devices (Synology, QNAP, TrueNAS), default credentials on routers, open Plex servers, vulnerable IoT devices, and anything running on common ports.</p>

<p>When Xfinity's threat intelligence recognizes a known-bad IP hitting your public address, it generates an alert. The key phrase in the notification? "was blocked." The probe never reached my TrueNAS—it hit the gateway's firewall and died.</p>

<h2 class="section-heading">The Bot and AI Factor</h2>

<p>What's interesting is the sheer volume. These aren't script kiddies manually probing networks. This is automated infrastructure—likely AI-assisted in target selection and vulnerability identification. The economics are simple: spin up disposable VPS instances for $5-10/month, run automated scans against residential IP ranges, flag anything that responds, sell access or exploit directly, burn the IP when it gets blacklisted, repeat.</p>

<p>The barrier to entry has never been lower. Tools like Shodan and Censys have legitimate uses, but they've also created a roadmap for attackers. Everyone knows where the NAS devices are.</p>

<h2 class="section-heading">Takeaways</h2>

<p><strong>Don't panic.</strong> Blocked attempts are informational—your defenses worked.</p>

<p><strong>Verify your exposure.</strong> Run a port scan against your public IP. If everything's closed, you're fine.</p>

<p><strong>Check for UPnP.</strong> Disable it on your router. Applications love to open ports without asking.</p>

<p><strong>Use Tailscale or similar.</strong> Keep your services off the public internet entirely.</p>

<p><strong>Restrict SMB/NFS.</strong> If you must have file shares, bind them to Tailscale IPs only.</p>

<p><strong>Enable auth everywhere.</strong> Even internal services should require authentication.</p>

<p>The scanners will keep coming. That's just the internet now. The goal isn't to make them stop—it's to make sure they find nothing when they arrive.</p>