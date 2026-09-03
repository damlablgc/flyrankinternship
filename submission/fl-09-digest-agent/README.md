# Daily Priority Digest Agent

## What it does

Reads unread Gmail messages from the last 24 hours, classifies each one by priority using the Gemini API, and sends a triaged summary to Telegram. Built for anyone who wants inbox triage without opening Gmail every morning.

The agent does not draft replies, create events, or take any action on the inbox — it only summarizes and classifies.

## Who it's for

Anyone with a Gmail inbox who wants a daily filter for what actually needs attention, versus what can wait. Built originally to triage internship and coursework emails against unrelated noise (newsletters, marketing, social notifications).

## Architecture

```
Schedule Trigger (runs daily)
      |
      v
Get many messages (Gmail node, last 24h, unread)
      |
      v
Code node (builds Gemini request body as a JS object)
      |
      v
HTTP Request (Gemini API, generateContent, Query Auth)
      |
      v
Send a text message (Telegram node)
```

The Code node builds the Gemini request body as a JavaScript object and serializes it with `JSON.stringify` before sending. Raw string interpolation of email content directly into the HTTP Request body was tested and rejected: unescaped quotes and newlines in email subjects/snippets silently broke the JSON payload.

## Setup

Requirements: Node.js (LTS, 20 or 22), a Gmail account, a Google Cloud project, a Gemini API key, a Telegram bot.

**1. Install and run n8n locally**

```powershell
npx n8n
```

Open `http://localhost:5678` in your browser and complete the one-time owner setup.

**2. Known network issue — read this first**

On some networks (observed on a Turkish home ISP), Node's TLS handshake over IPv6 fails with `EPROTO ... invalid session id` when n8n tries to reach Google's APIs. If you hit this error on the Gmail or HTTP Request node, force IPv4 before starting n8n:

```powershell
$env:NODE_OPTIONS = "--dns-result-order=ipv4first"
npx n8n
```

This is a network-layer issue, not an n8n or credential problem — confirmed by testing with `curl` (succeeds over IPv6) versus a plain Node `https.get()` call (fails with the same error), and resolved by forcing IPv4 resolution.

**3. Import the workflow**

In the n8n editor: **Add workflow** → menu (⋯) → **Import from File** → select the exported workflow JSON. Node connections and structure import correctly; credentials do not — you set those up next.

**4. Gmail credential (OAuth2)**

- Go to [Google Cloud Console](https://console.cloud.google.com), create a new project.
- Enable the **Gmail API** (APIs & Services → Library).
- Configure the **OAuth consent screen**: External user type, add your own Gmail address as a **Test user** (required — the app runs in Testing mode, unverified apps only allow listed test users to sign in).
- Create an **OAuth Client ID** (Web application type). Add this exact redirect URI:
  ```
  http://localhost:5678/rest/oauth2-credential/callback
  ```
- In n8n, open the **Get many messages** node → create new credential → paste the Client ID and Client Secret → sign in with Google.

**5. Gemini credential (Query Auth)**

- Get an API key at [Google AI Studio](https://aistudio.google.com/apikey).
- In n8n, open the **HTTP Request** node → Authentication: **Generic Credential Type** → Generic Auth Type: **Query Auth**.
- Create new credential: parameter name `key`, value = your API key.

**6. Telegram credential**

- In Telegram, message **@BotFather** → `/newbot` → follow the prompts → copy the bot token.
- Get your chat ID: message your new bot once, then visit `https://api.telegram.org/bot<TOKEN>/getUpdates` and read the `chat.id` field from the response.
- In n8n, open the **Send a text message** node → create new credential → paste the bot token → set the Chat ID field.

**7. Run it**

Trigger the workflow manually in the n8n editor, or wait for the schedule trigger. Confirm a digest arrives in Telegram.

## Usage example

Input: unread Gmail messages from the last 24 hours, including internship updates, newsletters, and calendar-adjacent emails.

Output (Telegram message):

```
NEEDS ATTENTION TODAY:
- FlyRank live guest session invite, relevant to internship schedule

COMING UP next 24-48h:
- Nothing in this category today.

CAN WAIT:
- LinkedIn notification, not actionable
- n8n onboarding emails, informational only
- Trendyol order update, unrelated to work/school
```

## v2 eval results

No labeled evaluation dataset exists for this agent's classification output — this is a known gap, stated openly below rather than glossed over.

What was verified through manual testing:
- End-to-end run completes successfully across repeated manual triggers (Gmail → Code → Gemini → Telegram), once the IPv6 network issue was worked around.
- One real classification bug was found and fixed during testing: the priority prompt originally matched sender domain `flyrank.ai`, but actual internship emails come from `flyrank.com`. This silently misrouted relevant emails into the "can wait" bucket. Fixed by updating the domain match in the Code node's system prompt.
- No systematic precision/recall measurement was run against a labeled set of emails. This would be the natural next step for a v3 eval.

## Limitations

- **No labeled evaluation set.** Classification quality was checked by manual spot-review of outputs, not measured against ground truth.
- **Gemini API 503 errors.** The model occasionally returns "currently experiencing high demand" errors, observed repeatedly during development. Mitigated with n8n's built-in Retry on Fail (multiple attempts with backoff on the HTTP Request node), but success is not guaranteed — this workflow depends on a third-party API's availability, outside this project's control.
- **Domain-matching is manual and brittle.** The prompt's sender-domain rules (e.g. `flyrank.com`, university domains) are hardcoded strings. A new relevant sender domain requires manually updating the prompt.
- **No calendar integration.** The current scope only reads Gmail; calendar events are not included (noted directly in the prompt as out of scope for this MVP).
- **Local-only.** This demo runs on a local n8n instance (`npx n8n`), not deployed to a persistent server — the schedule trigger only fires while the local process is running.

## Built with AI

This project was built with Claude as a development assistant: debugging the Node.js/TLS network issue (IPv6 handshake failures), diagnosing and fixing the OAuth redirect URI and Google Cloud Console setup, structuring the n8n Code node's JSON body construction, and drafting this README. I ran and verified every step myself — the OAuth flow, the credential setup, the end-to-end test runs, and the domain-matching bug were all found and confirmed through my own testing, not assumed from AI output.
