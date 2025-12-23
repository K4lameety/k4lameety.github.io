---
title: University CTF 2025
date: 2025-12-22
category: CTF
tags: [Blogs, Writeup, CTF, Forensic, Web]
description: "Writeup for University CTF 2025 by K4lameety."
---

![Banner](https://s3.eu-central-1.amazonaws.com/htb-ctf-prod-public-storage/ctf/banners/MqUyTLSOoOQppE1kWrgVBhQVTymEs9N0jhlTPc5q.jpg)
# 🎄 Grand Legend — The Tinsel Trouble of Tinselwick

In the snow-glittered village of Tinselwick, where peppermint chimneys puff cinnamon steam and toy trains zip between rooftops, the Festival of Everlight is the most magical night of the year. It's when the Great Snowglobe atop Sprucetop Tower shines brightest, sending cheer across the land and granting every child one heartfelt wish.

But this year… something's gone adorably wrong.

The Snowglobe's glow has flickered. The Wish-Wires have tangled. The Nutcracker Choir is singing in reverse. And strangest of all, a mischievous force known only as The Gingerbit Gremlin has stolen the Starshard Bauble, the ornament that powers the whole Festival!

With the countdown to Everlight Eve ticking fast, eight unlikely helpers—toy-fixers, cocoa-brewers, misfit scouts, and jolly engineers—must rally their sleighs, tighten their scarves, and follow peppermint-crumb trails through snowdrift mazes, puzzle cottages, and gingerbread vaults to recover the stolen magic.

This isn't a battle to save the world—it's a dash to save the spirit of the season.

The Great Snowglobe must sparkle again. And in Tinselwick, even the tiniest heart can outshine the longest night.

## Introduction

Hello, In this writeup, I will discuss solutions for 3 challenges that I successfully solved as a contribution to my team in this event. Starting from the **Forensic**, **Web**, and **Reversing** categories. I will explain them in a very simple way so they are easy to understand.

## Challenges

### Forensic: A Trail of Snow & Deception

- **Difficulty:** Easy  
- **Points:** 1000

### Description & Scenario

> Oliver Mirth, Tinselwick's forensic expert, crouched by the glowing lantern post, tracing the shimmerdust trail with a gloved finger. It led into the snowdrifts, then disappeared, no footprints, no sign of a struggle. He glanced up at the flickering Snowglobe atop Sprucetop Tower, its light wavering like a fading star. "Someone’s been tampering with the magic," Oliver murmured. "But why?" He straightened, eyes narrowing. The trail might be gone, but the mystery was just beginning. Can Oliver uncover the secret behind the fading glow?

### Challenge Questions

1. What is the Cacti version in use? (e.g. 7.1.0)
2. What is the set of credentials used to log in to the instance? (e.g., username:password)
3. Three malicious PHP files are involved in the attack. In order of appearance in the network stream, what are they? (e.g., file1.php,file2.php,file3.php)
4. What file gets downloaded using curl during exploitation process? (e.g. filename)
5. What is the name of the variable in one of the three malicious PHP files that stores the result of the executed system command? (e.g., $q5ghsA)
6. What is the system machine hostname? (e.g. server01)
7. What is the database password used by Cacti? (e.g. Password123)


### 1. Cacti Version

**Approach:** Search for "Cacti" string in packet details, then follow the HTTP stream.

**Steps:**
1. Filter packets containing "Cacti" → Found **Packet 357**
2. Follow HTTP Stream → Shows the version in response

![Packet 357](https://i.ibb.co.com/p6yHhk34/Screenshot-2025-12-22-170000.png)
![Version cacti](https://i.ibb.co.com/BHYwr7Vc/Screenshot-2025-12-22-170829.png)

**Answer 1:** `1.2.28`


### 2. Login Credentials

**Approach:** Find the login request (POST to index.php) to get username and password.

**Steps:**
1. Use Wireshark filter: `http.request.method == POST && http.request.uri contains "index.php"`
2. Analyze packet → Extract credentials from POST data

**Answer 2:** `marnie.thistlewhip:Z4ZP_8QzKA`


### 3. Web Shell & Payload Analysis

**Approach:** Find all PHP files uploaded by attacker, in order of appearance.

**Steps:**
1. Filter: `http.request.method == GET and http.request.uri contains ".php"`
2. Note the order files appear in network stream
3. These are the malicious PHP backdoors

![Version cacti](https://i.ibb.co.com/8LkrKNjL/Screenshot-2025-12-22-180639.png)

**Answer 3:** `JWuA5a1yj.php,ornf85gfQ.php,f54Avbg4.php`

---

### 4. Downloaded File

**Approach:** Find what file was downloaded using curl.

**Filter:** `http.request.method == GET and http.user_agent contains "curl"`

**Answer 4:** `bash`

---

### 5. PHP Variable Name

**Approach:** Decode the PHP file and find the variable storing command output.

**Steps:**
1. Decode Base64 PHP code from the web shell
2. Find `shell_exec()` function call
3. Identify variable that stores the output

**Key Code Line:**
```php
$a54vag = shell_exec($A4gVaXzY);  // ← This stores command output
```

**Answer:** `$a54vag`

---

### 6. System Hostname

**Approach:** Find encrypted hostname output, then decrypt using AES-256-CBC.

**Steps:**
1. Search for packet with `hostname` command execution
2. Find encrypted response in packet details
3. Use CyberChef with:
   - **Algorithm:** AES Decrypt
   - **Key:** `kF92sL0pQw8eTz17aB4xNc9VUm3yHd6G`
   - **IV:** `pZ7qR1tLw8Df3XbK`
   - **Input format:** Base64
   - **Output format:** UTF8

**Answer:** `tinselmon01`

---

### 7. Database Password

**Approach:** Find encrypted database config file output, then decrypt it.

**Steps:**
1. Search for packet with `cat include/config.php` command
2. Extract encrypted response
3. Decrypt using same AES-256-CBC key and IV as Question 6
4. Parse the config file to find `password` field

**Decryption Details:**
- **Key:** `kF92sL0pQw8eTz17aB4xNc9VUm3yHd6G`
- **IV:** `pZ7qR1tLw8Df3XbK`
- **Algorithm:** AES-256-CBC

**Answer:** `zqvyh2fLgyhZp9KV`

### Web: Silent Snow
### Reversing: CloudyCore

## Summary