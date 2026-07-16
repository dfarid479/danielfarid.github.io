---
layout: post
title: "Building a Hash Lookup Tool with SQLite — A Practical Application of Classroom Learning"
date: 2026-05-30 15:30:00 -0500
categories: forensics tools sqlite dfir
background: '/img/posts/sqlite.png'
---
<p>{% include ai-disclaimer.html %}</p>

<p>One of the things I genuinely enjoy about working in digital forensics is that the stuff you study and the stuff you actually need at work tend to collide in useful ways. This week was a good example of that.</p>

<p>I'm currently working through two separate tracks at the same time — an advanced mobile forensics course that covers SQLite databases in depth, and Domain 8 of the CISSP, which touches on the same subject from a security architecture angle. SQLite kept coming up in both, and I kept thinking: I should actually build something with this instead of just reading about it.</p>

<p>The problem I had in front of me was straightforward. I needed a way for myself and a few colleagues to quickly check image hash values against a law enforcement hash set — a JSON file containing over 18 million records. The existing solution was a browser-based tool that tried to load the entire file into memory on every session. It was painfully slow, crashed regularly, and honestly just wasn't practical for day-to-day use.</p>

<p>My first instinct was a Bash script. Parse the JSON, grep for the hash, done. But the more I thought about it, the more that felt like the wrong tool. Grep on an 11 GB JSON file every time you need a result isn't fast — it's just deferred slowness. And it offered nothing in terms of structure, metadata, or scalability.</p>

<p>SQLite clicked immediately as the right answer. Convert the JSON once, index it properly, and every subsequent lookup is a sub-millisecond binary search on a B-tree. That's the whole point of a database.</p>

<p>The build pipeline ended up being a Python script that memory-maps the source JSON and scans it with compiled regex — no full file load into RAM, no JSON parser bottlenecks. It makes two passes: one to collect series name metadata, and one to insert records in bulk with that metadata already attached. For 18.7 million records it builds the indexed SQLite database in under 15 minutes on a standard workstation SSD. The resulting database is 1.9 GB, down from the original 11 GB JSON, and every lookup hits an indexed BLOB column.</p>

<p>The front end is a small Python HTTP server (no external dependencies — stdlib only) that serves a browser UI and handles lookup requests. Coworkers double-click a batch file, a browser window opens, they paste in up to 15 hashes and get results in under a second. Match results include the category classification and series identification where available.</p>

<p>What I found most satisfying wasn't the finished product — it was how directly the coursework applied. Understanding how SQLite stores data on disk, how B-tree indexes work, and what PRAGMA settings actually do at the page cache level made the difference between a tool that works and one that works well. Domain 8 frames these concepts in terms of system resilience and data integrity. The mobile forensics course shows you where SQLite lives in the real world. Building something practical with it tied both threads together in a way that reading alone never quite does.</p>

<p>If you're studying SQLite and looking for a hands-on project, find a large dataset you actually care about and make it queryable. Nothing reinforces the theory like watching a 15-second grep become a 2-millisecond indexed lookup.</p>
