# 7.6 Deployment - Class Guide

## Today's Agenda

| # | Activity | Duration |
|---|----------|----------|
| 1 | DNS & HTTPS — concept + commands | 30 min |
| 2 | Peer Interview — verbal Q&A | 40 min prep + 30 min interviews |
| 3 | Coding Interview — Kanban API on CodeSandbox | 30 min |

---

# Part 1 — DNS & HTTPS — How the Web Finds Your App

## What is DNS?

When you type `myapp.com` into a browser, your computer has no idea where that is. It only understands IP addresses — numerical addresses like `216.58.214.46`. DNS (Domain Name System) is the system that translates human-readable domain names into IP addresses.

Think of it as the internet's phone book. You look up a name, it gives you the number.

---

## How a DNS Lookup Works

When you visit a domain, this happens in order:

1. Your browser checks its **local cache** — has it looked this up recently?
2. If not, it asks your **ISP's resolver** (a DNS server your internet provider runs)
3. The resolver asks a **Root Name Server** — the top of the DNS hierarchy
4. The root server points to the **TLD Name Server** (e.g. `.com`, `.io`, `.dev`)
5. The TLD server points to your domain's **Authoritative Name Server** (managed by your registrar or DNS provider)
6. The authoritative server returns the **IP address**
7. Your browser connects to that IP and loads the page

This entire process typically takes **milliseconds** and the result is cached so it doesn't repeat on every request.

---

## DNS Records

DNS is not just one mapping — it's a set of records, each serving a different purpose.

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Maps a domain to an IPv4 address | `myapp.com → 216.58.214.46` |
| **AAAA** | Maps a domain to an IPv6 address | `myapp.com → 2001:db8::1` |
| **CNAME** | Aliases one domain to another | `www.myapp.com → myapp.com` |
| **MX** | Routes email to a mail server | `myapp.com → mail.google.com` |
| **TXT** | Stores text — used for verification | Domain ownership proofs |

For connecting a custom domain to a Render deployment, you will use either an **A record** or a **CNAME record**.

- Use a **CNAME** when pointing a subdomain like `www.myapp.com` to your Render URL
- Use an **A record** when pointing a root/apex domain like `myapp.com` to an IP address

---

## Connecting a Custom Domain to Render

### Step 1 — Buy a domain

Purchase a domain from a registrar. Some popular options:

- **Namecheap** — cheap first year, popular with devs
- **Cloudflare** — sells at cost price, no markup, best value long term
- **Squarespace Domains** (formerly Google Domains) — clean UI

Note: domains cost money (~$10–15/year for a `.com`). There is no free option.

---

### Step 2 — Add your domain in Render

Go to your Render service → **Settings** → **Custom Domains** → click **Add Custom Domain** → type your domain name.

Render will then show you exactly what DNS records to add. It looks like this:

| # | Hostname | Target value | Record type |
|---|----------|-------------|-------------|
| 1 | `www` | `your-app.onrender.com` | CNAME |
| 2 | `@` | `your-app.onrender.com` | CNAME |

**What do these mean?**

- **Hostname** is the address you're configuring:
  - `www` = `www.yourdomain.com`
  - `@` = the root domain itself — just `yourdomain.com` with nothing before it. Not `app.yourdomain.com` or `dashboard.yourdomain.com` — just the bare domain.
- **Target value** is where that address should point — your Render app URL

So in plain English:
> "When someone types `www.yourdomain.com` → send them to your Render app"
> "When someone types `yourdomain.com` → also send them to your Render app"

You need both records because visitors might type either version.

> **Note:** Some registrars don't support CNAME on the root domain (`@`). In that case, Render also provides an **A record** IP address (e.g. `216.24.57.1`) to use instead.

---

### Step 3 — Add the records in your registrar

Log into Namecheap (or wherever you bought the domain) → find **Advanced DNS** settings for your domain → add both records exactly as Render specified.

Each subdomain is its own separate record. If you later want `app.yourdomain.com` or `api.yourdomain.com` to work, you'd add a third and fourth record. Companies use this pattern all the time:

- `app.notion.so` → the main product
- `api.notion.so` → the API
- `www.notion.so` → marketing site

Each is a separate DNS record pointing to a different server.

---

### Step 4 — Verify and wait

Go back to Render and click **Verify**. Render will start checking for your records. On the same screen you'll see **Certificate Status: Waiting for Verification** — this is Render waiting to auto-provision your TLS certificate (HTTPS). It can't do that until DNS propagates.

Check propagation progress at [dnschecker.org](https://dnschecker.org) — it shows you which DNS servers around the world have picked up your new records yet.

---

## What is TLS / HTTPS?

HTTP sends data as plain text — anyone intercepting the connection can read it. **TLS (Transport Layer Security)** encrypts the connection between the browser and the server, which is what makes `https://` work.

When you see the padlock icon in your browser, TLS is active.

Render automatically provisions a **free TLS certificate** via Let's Encrypt for any custom domain you connect. You do not need to configure this manually.

---

## DNS Propagation

When you update a DNS record, the change does not take effect instantly worldwide. Different DNS servers cache records for different lengths of time, controlled by a setting called **TTL (Time To Live)** — measured in seconds.

A TTL of `3600` means DNS servers cache the record for 1 hour before checking for updates.

This is why after pointing your domain to a new server, some users may still see the old site for a while. Full propagation typically takes **a few minutes to 24 hours**.

---

## Try It — DNS & Network Commands

Open your terminal and try these. All work on Mac/Linux. Windows equivalents noted where different.

---

### Ping — is this server alive?
```bash
ping google.com
```
Sends packets to the server and measures how long a response takes (in milliseconds). Good first check when something isn't loading — is the server even reachable?

```bash
ping your-app.onrender.com
```

---

### nslookup — what IP does this domain resolve to?
Works on **Windows, Mac, and Linux**.
```bash
nslookup google.com
```

Look up a specific record type:
```bash
nslookup -type=MX gmail.com       # who handles Gmail's email?
nslookup -type=TXT google.com     # text/verification records
nslookup -type=CNAME www.github.com
```

Reverse lookup — what domain owns this IP?
```bash
nslookup 216.58.214.46
```

---

### dig — deeper DNS inspection (Mac/Linux)
```bash
dig google.com
```

Get just the answer, no noise:
```bash
dig google.com +short
```

Look up specific record types:
```bash
dig google.com MX
dig google.com TXT
dig www.google.com CNAME
```

Trace the full resolution path — watch DNS work step by step from root servers down:
```bash
dig google.com +trace
```
> This one is the most satisfying to run in class — you can see every hop in the DNS hierarchy.

Reverse lookup with dig:
```bash
dig -x 216.58.214.46 +short
```

---

### tracert / traceroute — how does traffic get there?
Shows every router (hop) your traffic passes through to reach a destination. Great for seeing that a request might travel through multiple countries before reaching a server.

**Mac/Linux:**
```bash
traceroute google.com
```

**Windows:**
```bash
tracert google.com
```

---

### whois — who owns this domain?
```bash
whois google.com
whois github.com
```
Shows registration info — registrar, owner (if not private), creation date, expiry date, nameservers. Useful for checking when a domain was registered or who to contact about it.

> Windows: `whois` may not be installed by default. Use [who.is](https://who.is) in the browser instead.

---

### curl — what does a server actually respond with?
```bash
curl -I https://google.com
```
`-I` fetches just the headers — you can see the HTTP status code, content type, server info, and whether it redirects.

```bash
curl -I https://your-app.onrender.com
```

Check if your app is returning `200 OK` or something else.

---

### Online tools (no install needed)
- [dnschecker.org](https://dnschecker.org) — check DNS propagation globally
- [who.is](https://who.is) — whois lookups in the browser
- [whatsmydns.net](https://whatsmydns.net) — see which DNS servers worldwide have picked up your record yet

---

## Key Takeaways

- DNS translates domain names to IP addresses
- Different record types serve different purposes — A, CNAME, MX, TXT
- Render handles TLS automatically — HTTPS is free and automatic
- DNS changes take time to propagate due to caching (TTL)
- You can inspect DNS records directly from the terminal with `nslookup` and `dig`

---

# Part 2 — Peer Interview

## How It Works

- You will be placed in pairs
- Each person has **two roles** — interviewer and interviewee
- You will each be interviewed on a different topic track:
  - **React track** — HTML, CSS, JavaScript, and React
  - **Express track** — HTML, CSS, JavaScript, and Express

## Your Prep Track

- If you are being **interviewed on React** → study React concepts for 40 minutes
- If you are being **interviewed on Express** → study Express concepts for 40 minutes
- You will **not** see the questions you'll be asked — you answer cold

## Interview Format

- The interviewer holds the question pack — they read each question aloud
- The interviewee answers verbally — no Googling, no notes
- The interviewer scores each answer using the rubric in their pack
- After both interviews, share your scores and give constructive verbal feedback

## Scoring

Each question is scored out of 4:

| Score | What it means |
|-------|--------------|
| 4 | Complete, confident, correct terminology |
| 3 | Mostly correct, minor gaps |
| 2 | Partial understanding |
| 1 | Attempted but largely incorrect |
| 0 | No answer or completely wrong |

Plus **10 points for soft skills** — communication, honesty about gaps, staying calm, giving examples.

---

# Part 3 — Coding Interview

## The Challenge

You will build a **Kanban Board REST API** using Express and MongoDB Atlas.

This is a real interview-style coding challenge — the kind you will encounter when applying for dev jobs.

## Getting Started

1. Open the CodeSandbox link shared in the class chat
2. Click **Fork** in the top right — **you must fork before editing**, otherwise everyone edits the same file
3. Add your **MongoDB Atlas connection string** to the `.env` file
4. Run `pnpm install` in the terminal
5. Run `pnpm start` to start the server
6. Implement the routes — follow the `TODO` comments in `index.js`



> ⚠️ **Fork first. Do not edit the original.**

> 🐛 **Heads up:** The skeleton may contain one or more intentional bugs. Part of the challenge is spotting and fixing them.

## What to Build

### POST /boards
- Accepts `{ "title": "..." }` in the request body
- Creates a new board item with an auto-incremented `id` starting from 1 and `stage` defaulting to 1
- Returns **status 201** with the created item

### PUT /boards/:id
- Accepts `{ "stage": 2 }` in the request body
- If stage is not 1, 2, or 3 — return **status 400**
- If valid — update the item and return **status 200** with the updated item

## Testing Your API

Use **Postman**, **Thunder Client**, or curl to test your endpoints as you build.

```bash
# Create a board item
curl -X POST http://localhost:3000/boards \
  -H "Content-Type: application/json" \
  -d '{"title": "My first task"}'

# Update stage
curl -X PUT http://localhost:3000/boards/1 \
  -H "Content-Type: application/json" \
  -d '{"stage": 2}'
```

> Windows users: use Git Bash or Postman instead of CMD for curl commands.

## Scoring

| Criteria | Points |
|----------|--------|
| POST returns 201 with correct shape | 25 |
| POST auto-increments id correctly | 25 |
| PUT returns 400 for invalid stage | 20 |
| PUT returns 200 with updated item | 20 |
| Clean, readable code | 10 |
| **Total** | **100** |

## Bonus

If you finish early, implement `GET /boards` — returns all board items as an array ordered by `id`.
