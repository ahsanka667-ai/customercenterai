# CustomerCenter AI — Product & Go-to-Market Plan

**AI order confirmation for Facebook-first and website sellers**

Version 0.1 · 23 October 2026 · Working draft

---

## 0. The short answer

**Facebook is not a phone channel.** There is no Meta API that lets your app call a customer. Messenger is a chat channel with strict rules about when you may message someone. So the product is really two layers:

1. **Intake layer** — turn website orders, Messenger chats, post comments, and Lead Ads into a structured order with a phone number and consent.
2. **Confirmation layer** — an AI voice agent calls that number, confirms the product/price/address, and writes back a clean outcome.

Build the confirmation engine **once**. Treat the website and Facebook as **adapters** around it.

- The website adapter is easy: you already control the order data and the phone number.
- The Facebook adapter is harder: OAuth, webhooks, App Review, 24-hour/7-day messaging windows, no phone number in the profile, and unstructured chat. But Facebook is where the orders are, so you cannot skip it.

**The sequence I recommend:**

> Build the voice engine + website/CSV intake first (2–3 weeks). Start Meta Business Verification and App Review in week 1, in parallel, because approval takes weeks. Add Messenger, comments, and Lead Ads once the voice engine is proven with real merchants.

**Do not build a generic "AI call center."** Build one workflow: **COD order confirmation in Bangla/English, within 5 minutes of the order, with address repair and a clean outcome for the merchant.** That is the wedge.

> **Assumption:** "product purchases" means consumer e-commerce orders, mostly cash-on-delivery (COD), from Facebook/Instagram sellers and website stores in Bangladesh. If you mean B2B distributor purchase orders, the same engine applies but the script, consent model, and integrations change — see §14.

### Plan at a glance

| Decision | Recommendation |
|---|---|
| **Beachhead market** | Bangladesh Facebook/Instagram COD sellers, 300–3,000 orders/month |
| **Wedge** | AI voice confirmation of COD orders within 5 minutes |
| **Core output** | Structured outcome: confirmed / cancelled / callback / wrong number / address fixed |
| **Channel strategy** | Voice out; Facebook is intake + in-window chat |
| **Build order** | Voice engine + website/CSV → pilot → Messenger/comments/Lead Ads |
| **Voice stack (first 1,000 calls)** | Vapi or Retell + Twilio/Plivo; local SIP later |
| **Facebook priority** | Messenger order extraction → Click-to-Messenger ads → comments → Lead Ads |
| **Pricing (hypothesis)** | ৳2,500/mo + ৳12–18/call; per-confirmed-order later |
| **Time to pilot** | 3 weeks |
| **Time to Facebook live** | 8–12 weeks, mostly App Review |
| **Biggest risk** | Unit economics vs. cheap human callers; local CLI and short calls are mandatory |

### Glossary

- **COD** — cash on delivery.
- **RTO** — return to origin; an undelivered parcel sent back to the seller.
- **PSID** — Page-Scoped ID; Meta's per-Page identifier for a user.
- **CTM** — Click-to-Messenger ad.
- **ASR** — automatic speech recognition (speech-to-text).
- **TTS** — text-to-speech.
- **CLI** — caller line identification; the number the customer sees.
- **DNC** — do-not-call list.
- **NDR** — non-delivery report from a courier.
- **AOV** — average order value.
- **3PL** — third-party logistics provider.

---

## Table of contents

1. [What we are building](#1-what-we-are-building)
2. [Why this market](#2-why-this-market)
3. [Customer and wedge](#3-customer-and-wedge)
4. [The two channels: website vs Facebook](#4-the-two-channels-website-vs-facebook)
5. [Facebook integration in detail](#5-facebook-integration-in-detail)
6. [End-to-end flows](#6-end-to-end-flows)
7. [The AI call](#7-the-ai-call)
8. [Architecture](#8-architecture)
9. [Unit economics and pricing](#9-unit-economics-and-pricing)
10. [Go-to-market](#10-go-to-market)
11. [Roadmap](#11-roadmap)
12. [Metrics and proof](#12-metrics-and-proof)
13. [Risks](#13-risks)
14. [Open decisions](#14-open-decisions)
15. [What to build first in this repo](#15-what-to-build-first-in-this-repo)
16. [Sources](#16-sources)
17. [Appendix A — Bangla call script](#appendix-a--bangla-call-script)
18. [Appendix B — Facebook App Review checklist](#appendix-b--facebook-app-review-checklist)
19. [Appendix C — API payloads](#appendix-c--api-payloads)
20. [Appendix D — 30-day validation plan](#appendix-d--30-day-validation-plan)

---

## 1. What we are building

### 1.1 One sentence

> **CustomerCenter AI calls every new cash-on-delivery order within minutes, confirms the product, price, and address in natural Bangla, and tells the merchant what to ship — before the courier is booked.**

### 1.2 The job to be done

A Facebook seller gets an order in Messenger, a comment, a lead form, or a website checkout. The order has a phone number and an address, but the seller does not know:

- Is this a real order, or an impulse/fake order?
- Is the phone number real and reachable?
- Is the address complete enough for the courier?
- Is the customer actually going to accept the parcel and pay cash?
- Should we ship it, hold it, or cancel it?

Today the seller either calls manually, sends a WhatsApp/Messenger message, or ships blindly. The result is **RTO (return to origin)**: the parcel comes back, the seller pays courier charges both ways, and the product is tied up for weeks.

Our job is to turn a messy order into a **confirmed, dispatch-ready order** with a structured outcome.

### 1.3 What the merchant actually buys

| Value | How we deliver it |
|---|---|
| Fewer returns (lower RTO) | Confirm intent before dispatch; cancel bad orders early |
| Faster dispatch | Call within 5 minutes, not next day |
| Fewer failed deliveries | Repair address, landmark, delivery time window |
| More prepaid orders (optional) | Offer a small prepaid discount on the call |
| Higher order value (optional) | Add a relevant upsell on the call |
| Labor savings | Replace/assist manual confirmation callers |
| Visibility | Recording, transcript, outcome, reason codes, dashboards |
| Consistency | Same script, same questions, every order, 24/7 |

The merchant is not buying "AI" or "minutes." They are buying **fewer lost parcels and a faster, cleaner dispatch process**.

### 1.4 What we are not (v1)

- Not an inbound customer support desk.
- Not a general sales chatbot.
- Not a Messenger broadcast tool.
- Not a replacement for complex human sales calls.
- Not a Shopify App Store product yet.
- Not a call center with 50 agents.

One workflow, done well, is the whole MVP.

---

## 2. Why this market

### 2.1 The COD problem is the wedge

Bangladesh is a cash-on-delivery market. Industry estimates put COD at roughly **70–80% of e-commerce orders**, and COD orders return at **20–30%** versus low single digits for prepaid orders. Every RTO costs the seller:

- courier charge both ways (typically ৳120–200 depending on weight/zone),
- packaging and handling,
- capital tied up for weeks,
- possible product damage or seasonality loss,
- a lost sale that may never be recovered.

The highest-leverage fix is **before dispatch**: confirm the order, confirm the address, and confirm that someone will be there with cash. That is exactly the job an AI voice agent can do at scale.

### 2.2 Why Facebook is the right battlefield

In Bangladesh, a huge share of social commerce happens on Facebook Pages and Messenger, not on a Shopify checkout:

- Customers ask "দাম কত?" in comments and then order in Messenger.
- Sellers collect name, phone, address, and product details in chat.
- There is no reliable checkout, no order API, and no phone number in the Facebook profile.
- Orders are unstructured text, voice notes, and screenshots.

This is painful for sellers, which means **the intake layer itself is valuable**. A tool that reads the chat, extracts the order, and then calls to confirm is much more useful than a generic dialer.

Meta also gives you intent signals a website-only tool cannot see:

- **Messenger conversations** — high intent, already talking to the seller.
- **Post comments** — "price?", "inbox", "order korte chai" — high intent, public.
- **Lead Ads** — name + phone + form answers, ready to call.
- **Click-to-Messenger ads** — paid traffic that opens a chat window of up to **7 days** (since 28 September 2026). This is the best Facebook entry point for a confirmation workflow in BD.

### 2.3 Why now

1. **Bangla voice AI is finally usable.** ElevenLabs supports Bengali on v3/v4, Azure has `bn-BD` voices (Nabanita, Pradeep), Google has `bn-BD` STT (Chirp 3, preview) and Gemini-TTS `bn-BD`.
2. **Telephony APIs cover Bangladesh outbound.** Twilio lists ~$0.060/min, Plivo ~$0.046/min. (Local SIP/CLI is still the hard part — see §9.4.)
3. **Meta tightened the rules.** Message tags were deprecated in April 2026; generic broadcast tools are being forced to become compliant workflow tools. That is an opening for a product that respects the windows and still gets the job done.
4. **Sellers are under margin pressure.** RTO is a board-level metric for e-commerce brands, and Facebook sellers feel it every week.

---

## 3. Customer and wedge

### 3.1 Ideal customer profile (ICP)

**Primary:** Bangladesh Facebook/Instagram sellers in fashion, cosmetics, home, and gadgets with:

- 300–3,000 COD orders/month,
- 3–20 operations staff,
- orders coming through Messenger, comments, and phone,
- shipping with Pathao, Steadfast, RedX, or Paperfly,
- currently confirming by phone/WhatsApp or not at all,
- a painful RTO number (25%+).

**Secondary:**

- Website/WooCommerce/Shopify brands with COD and a real order API.
- Agencies managing 10–50 Facebook Pages (white-label opportunity).
- Couriers/3PLs who want to offer confirmation as a value-added service to their merchants.

**Not first:**

- Enterprise brands with an existing 50-seat call center (long sales cycle, procurement).
- Pure prepaid stores (low RTO, less pain).
- Very low-AOV impulse sellers (call cost is too high relative to margin).

### 3.2 The wedge

> **COD order confirmation for Facebook-first sellers.**

One workflow. One outcome. One dashboard. Expand later into delivery scheduling, NDR recovery, prepaid conversion, upsell, and inbound support.

### 3.3 Positioning

- **For sellers:** "Confirm every COD order in 5 minutes — in Bangla — before you ship."
- **For agencies:** "White-label AI confirmation for all your client Pages."
- **For couriers:** "Cut RTO for your merchants without adding call-center headcount."

Avoid these phrases:

- "AI call center" — commodity, scares call centers, invites comparison on price per minute.
- "Automate all customer support" — too broad, impossible to prove.
- "Message all your customers on Messenger" — violates Meta policy and gets apps restricted.

---

## 4. The two channels: website vs Facebook

### 4.1 Website (the easy adapter)

**Flow:**

1. Checkout or order API sends order + phone + consent to us.
2. We call within 5 minutes.
3. We write the outcome back via webhook.
4. Merchant ships, holds, or cancels.

**Why it is easy:** you control the order data, the phone number, and the consent language. There is no Meta review, no messaging window, no unstructured chat.

**What is still hard (shared with Facebook):**

- telephony and caller ID,
- Bangla speech recognition,
- answer rates,
- call consent and recording disclosure,
- cost per call.

**Integrations, in order of effort:**

| Integration | Effort | When |
|---|---|---|
| CSV upload | 1–2 days | Week 1 |
| REST API + HMAC webhook | 2–3 days | Week 1 |
| WooCommerce plugin | 1 week | Week 4–5 |
| Shopify app | 3–4 weeks + review | Month 3+ |
| Custom "Confirm order" JS snippet | 1 week | Month 2 |

### 4.2 Facebook (the hard adapter, but where the orders are)

There are **four Facebook entry points**, plus a website-to-Messenger bridge. They behave differently. This is the part most people get wrong.

| Entry point | What you get | Messaging window | What to do |
|---|---|---|---|
| **Messenger DM / organic chat** | PSID, message text, attachments | 24 hours from the user's last action | Chat, collect phone + address, create order, call, send confirmation card |
| **Click-to-Messenger ad** | PSID + ad referral | Up to **7 days** from the first message after the ad click | Same as above, with more room to converse |
| **Page post comment** | Commenter ID, comment text, post ID | One **private reply** within 7 days; then 24h after the person replies | Reply privately, move to Messenger, collect phone, call |
| **Lead Ads form** | Name, phone, email, custom answers; no PSID | No chat window; phone call is separate | Fetch lead via `leadgen` webhook, call in <5 minutes, update CRM |
| **Website + Send to Messenger / checkbox plugin** | Links a website session to a PSID + opt-in | Opens a messaging window | Optional: let website buyers opt into Messenger updates |

**What Facebook does not give you:**

- ❌ No API to place an outbound phone call.
- ❌ No phone number from the user's profile.
- ❌ No cold DMs outside the allowed window.
- ❌ No unlimited follow-ups.
- ❌ No guarantee of delivery, open, or read.
- ❌ No way to "just call" a commenter.

**So the product must capture the phone number first, then use telephony for the actual confirmation call.** Facebook is the **intake + in-window chat** layer; voice is the **reliable outbound** layer.

**Why not WhatsApp first?** WhatsApp Business API is a strong channel in Bangladesh, but it is template-based outside the 24-hour service window, charges per delivered template, and requires opt-in. It is a good **phase 3** addition (order updates, missed-call follow-up, prepaid links), not the wedge. Facebook is where the product discovery and ordering behavior already happens for this ICP.

### 4.3 The policy rules that shape the product

These are not fine print. They determine what the product can promise.

| Rule | What it means | Product implication |
|---|---|---|
| **Standard messaging window** | 24 hours from the user's action; up to 7 days for the first message after a Click-to-Messenger / Click-to-Direct ad | Track the window per PSID; choose message type programmatically |
| **Message tags deprecated** | `CONFIRMED_EVENT_UPDATE`, `ACCOUNT_UPDATE`, `POST_PURCHASE_UPDATE` return error 100 since 27 April 2026 | Do not build on tags; use utility templates or Marketing Messages |
| **Utility templates** | Order/account updates and appointment reminders; no marketing; template approval; `page_utility_messaging` permission; `messaging_type=UTILITY` | Use for order status outside the window, not for promotional follow-ups |
| **HUMAN_AGENT tag** | 7 days, **human-written only**, support use cases; App Review required | Never let the bot send with HUMAN_AGENT; route to a human if needed |
| **Private replies** | One message within 7 days of a comment | Use once to move the commenter into Messenger; do not spam |
| **Marketing Messages API** | Paid, opt-in, outside the window; separate API | Later, if ever; not needed for confirmation |
| **Responsiveness** | Automated bots must respond to user input within 30 seconds | Chat agent must be fast and always reply, even with "let me check" |
| **Lead data retention** | Lead data available for 90 days | Fetch and store only what you need; delete per policy |
| **App Review + Business Verification** | Required for advanced access to messaging/lead permissions | Start in week 1; it is the long pole |

**The single most important product decision:** **voice is the outbound channel, Facebook is the intake channel.** If you build the product the other way around — assuming you can message anyone on Messenger whenever you want — Meta will restrict the app and your merchants will lose access.

### 4.4 What the Facebook product actually looks like

Three surfaces, built in this order:

1. **Inbox assistant (read + reply + extract order).**
   - Receives Messenger webhooks.
   - Discloses that it is an automated assistant at the start of the conversation (Meta recommends this; some jurisdictions require it).
   - Answers common questions (price, delivery, stock) from the merchant's catalog (CSV/Google Sheet/WooCommerce sync in v1).
   - Collects name, phone, address, product, variant, quantity.
   - Asks permission to use the phone number for the confirmation call.
   - Creates a structured order and asks for confirmation.
   - Sends a "We'll call you in 2 minutes to confirm" message while the window is open.
   - Then triggers the voice call.

2. **Comment-to-order.**
   - Receives `feed` webhooks for comments on the Page's posts.
   - Sends one private reply with a Messenger link.
   - Once the person replies, the 24-hour window opens and the inbox assistant takes over.

3. **Lead Ads fast-call.**
   - Receives `leadgen` webhooks.
   - Fetches the lead via Graph API.
   - Validates the phone number.
   - Calls within 5 minutes (speed-to-lead matters).
   - Writes the outcome back to the CRM/webhook.

**Do not start with a full inbox replacement.** Start with order extraction + confirmation. The merchant keeps their existing inbox tool if they want; we plug into the order flow.

---

## 5. Facebook integration in detail

### 5.1 What you need before writing code

- **Meta Business Portfolio** (Business Manager) for your company.
- **Business Verification** — legal entity name, address, phone, business documents. Start now; it can take days to weeks.
- **Meta app** of type **Business**.
- **Privacy policy** URL that explicitly covers Facebook Page data, Messenger messages, lead data, retention, and deletion.
- **Terms of service**.
- **Data deletion callback** URL and instructions.
- **Deauthorize callback** URL.
- **HTTPS webhook** with a valid certificate and a fast 200 response.
- **A test Facebook Page** with real conversations and a test lead form.

**Pilot shortcut if App Review is slow:** connect the merchant's Page to an existing Messenger automation platform (ManyChat/Chatfuel) and forward new-message events to us via Zapier/Make. This is a temporary bridge for the first 1–2 pilots — not the core architecture. It adds cost, latency, and a dependency, and it may not give you raw webhook access. Use it to learn, then replace it with your own Meta app.

### 5.2 Permissions to request (start minimal)

| Permission | Why | When |
|---|---|---|
| `pages_show_list` | Let the merchant choose which Page to connect | Phase 1 |
| `pages_manage_metadata` | Subscribe the app to Page webhooks | Phase 1 |
| `pages_messaging` | Read/send Messenger messages | Phase 1 |
| `pages_read_engagement` | Read comments/Page content for comment-to-DM | Phase 1 |
| `page_utility_messaging` | Send utility templates outside the 24h window | Phase 2 |
| `leads_retrieval` | Read Lead Ads form submissions | Phase 2 |
| `pages_manage_ads` / `ads_management` | Lead attribution and ad-level reporting | Phase 2 |
| `instagram_basic`, `instagram_manage_messages` | Instagram DM (same policy family) | Phase 3 |

**Rule:** request only what the demo needs. Every extra permission is another rejection risk and another data-retention obligation.

### 5.3 OAuth and Page connection

1. Merchant clicks **Connect Facebook Page**.
2. Facebook Login for Business (or standard Facebook Login) returns a user token with the requested scopes.
3. Exchange for a long-lived user token, then fetch Page tokens via `GET /me/accounts`.
4. Store the Page token **encrypted** (KMS or app-level envelope encryption) with the Page ID, scopes, and connection status.
5. Subscribe the app to the Page:
   `POST /{page-id}/subscribed_apps` with `subscribed_fields=messages,messaging_postbacks,messaging_optins,message_reactions,feed,leadgen`.
6. On deauthorize or token error (`code 190`), mark the connection as disconnected and ask the merchant to reconnect.

**Never** put Page tokens in the frontend, logs, or error messages.

### 5.4 Webhook receiver

**Verification (GET):**

```
GET /webhooks/meta?hub.mode=subscribe&hub.verify_token=...&hub.challenge=...
→ 200 with hub.challenge if the token matches
```

**Events (POST):**

1. Read the **raw body** (do not parse before signature verification).
2. Verify `X-Hub-Signature-256` = `sha256(app_secret, raw_body)` using a timing-safe compare.
3. Return `200` immediately (under 5 seconds).
4. Enqueue the event; process asynchronously.
5. Deduplicate by `message.mid`, `leadgen_id`, or a hash of the event.
6. Log the event with tenant, Page ID, type, and status for debugging.

**Why this matters for App Review:** reviewers test the webhook live. A timeout, a 404, or a failed handshake is a rejection.

### 5.5 Message window state machine

For every PSID, store:

- `last_user_message_at`,
- `window_source` (`organic`, `ctm_ad`, `plugin`, `comment_reply`),
- `window_expires_at` (24h or 7d),
- `last_bot_message_at`,
- `opted_out`,
- `human_takeover`.

Before sending, compute the allowed message type:

| Window state | Allowed | Not allowed |
|---|---|---|
| Open (24h/7d) | `RESPONSE` or `UPDATE`, promotional content allowed | Spam, harassment |
| Closed, utility need | Approved utility template with `UTILITY` | Marketing content, free-form promo |
| Closed, human support | `HUMAN_AGENT` written by a human | Bot messages, sales follow-ups |
| Closed, marketing | Marketing Messages API (paid, opt-in) | Free-form messages |

Build this as a **single guard function** used by every send path. Do not let feature teams call the Send API directly.

### 5.6 App Review: what actually gets approved

Meta reviewers are looking for a **working, end-to-end use case**, not a settings page. Prepare:

- A **3–5 minute screencast**, narrated, no cuts:
  1. Log in with Facebook.
  2. Select the test Page.
  3. Grant the permission.
  4. Send a message from a test user to the Page.
  5. Show the message arriving in your app.
  6. Create an order from the conversation.
  7. Trigger the call (or show the call record).
  8. Show the outcome in the dashboard.
  9. Show the webhook receiving a lead (if requesting `leads_retrieval`).
  10. Show the data deletion flow.
- A **privacy policy** that names Messenger, Page comments, Lead Ads, retention periods, and deletion.
- A **data deletion callback** that returns a confirmation URL and code.
- A **test Page** with real content and a reviewer test account.
- **Permission justifications** written in plain language: "We use `pages_messaging` to reply to customers who message the Page about an order, and to send order confirmation updates."
- **No fake data** in the demo. Reviewers can tell.

**Timeline:** plan for **2–6 weeks**, and assume at least one rejection. Submit early; do not block the pilot on it.

### 5.7 Data handling

- Store only the fields you need: PSID, name, phone, address, order, messages relevant to the order.
- Encrypt phone numbers and addresses at rest (field-level or envelope encryption).
- Recordings: private bucket, signed URLs, 90-day default retention, delete on request.
- Lead data: fetch within 90 days; do not retain beyond the stated purpose.
- Honor deletion requests across orders, calls, transcripts, and recordings.
- Publish a data processing addendum (DPA) for merchants.
- Train staff on PII access; log admin views.

---

## 6. End-to-end flows

### 6.1 Website COD order

```mermaid
sequenceDiagram
  participant C as Customer
  participant S as Store
  participant API as Our API
  participant V as Voice Agent
  participant D as Dashboard
  C->>S: Place COD order
  S->>API: POST /v1/orders (phone + consent)
  API->>API: Validate, dedupe, schedule
  API->>V: Call within 5 minutes
  V->>C: Bangla confirmation call
  V->>API: Outcome + transcript + recording
  API->>S: Webhook: confirmed / cancelled / callback
  API->>D: Live order + call record
```

**Timing targets:**

- Order received → call started: **< 5 minutes** (95th percentile < 10 min).
- Call length: **60–120 seconds**.
- Outcome available in dashboard/webhook: **< 30 seconds after hangup**.

### 6.2 Facebook Messenger order

1. Customer comments or DMs: "এই ড্রেসটার দাম কত?" / "order korte chai."
2. Bot replies in-window (or one private reply for a comment).
3. Bot collects: product, variant, quantity, name, phone, address.
4. Bot reads back the summary and asks "ঠিক আছে?"
5. Bot creates the order and sends: "ধন্যবাদ! অর্ডারটি কনফার্ম করতে ২ মিনিটের মধ্যে একটি কল আসবে।"
6. Voice agent calls.
7. Outcome is written back; if the window is open, send a confirmation card with buttons:
   - ✅ Order confirmed
   - 📍 Change address
   - ❌ Cancel order
8. Merchant dashboard updates; optionally auto-book courier.

**Why the pre-call message matters:** it raises answer rates, sets expectations, and gives the customer a chance to say "call me later" before you dial.

### 6.3 Lead Ad

1. `leadgen` webhook arrives with `leadgen_id`.
2. Fetch the lead: `GET /{leadgen_id}?access_token=...`.
3. Validate phone:
   - normalize to E.164 (`+8801[3-9]XXXXXXXX` for BD mobile),
   - check duplicates and existing orders,
   - check the tenant's DNC/suppression list,
   - optional carrier/lookup check for reachability.
4. Call within **5 minutes**.
5. Outcome webhook to the merchant's CRM.
6. If no answer: SMS + retry schedule.

**Speed-to-lead is the whole game.** A lead called in 5 minutes converts far better than one called next day. Make this a product promise and measure it.

### 6.4 No answer / retry policy

| Attempt | Timing | Action |
|---|---|---|
| 1 | Immediately | Call |
| 2 | +2 hours | Call; if no answer, SMS with order summary |
| 3 | Next morning (10–11am) | Call; if no answer, mark `unreachable` and notify merchant |
| Voicemail detected | On attempt 1 | Hang up, send SMS, retry per schedule |
| Opt-out / DNC | Immediately | Stop all calls; record consent withdrawal |

**Rules:**

- Call window: **9:00–21:00** local time, 7 days a week (configurable).
- No calls during major religious holidays unless the merchant opts in.
- Max 3 attempts per order.
- Never call a number that has asked not to be called.
- Pre-call SMS/Messenger notice where possible.

### 6.5 Human handoff

Transfer to the merchant's number when:

- the customer asks for a human,
- the customer is angry or threatening,
- the request is outside the script (custom order, bulk, refund, complaint),
- the AI confidence is low for two consecutive turns,
- the customer explicitly asks to change something the AI cannot change.

A warm transfer should pass context: customer name, order ID, issue, language, and a one-line summary. If no one answers, create a callback task and tell the customer when to expect it.

---

## 7. The AI call

### 7.1 Design principles

- **Short.** 60–120 seconds. The customer is not shopping; they are confirming.
- **Structured.** One job, a fixed set of slots, no open-ended sales pitch.
- **Confirm, don't persuade.** Read back the order; ask "ঠিক আছে?"
- **Repair data.** Address, landmark, delivery window, quantity, variant.
- **Detect "no" early.** If the customer says cancel, cancel politely. Do not argue.
- **Disclose.** Say it is an automated assistant and that the call is recorded (where required).
- **Fallback.** DTMF, SMS, human transfer, and a clear "we'll call you back."

### 7.2 The call state machine

```text
dial
 → answered? ──no──→ voicemail? → leave message / SMS → retry
     │yes
 → greeting + AI disclosure + permission to talk
 → identity check (name / order reference)
 → slot confirmation (product, variant, qty, amount)
 → address confirmation + repair
 → delivery preference (morning / afternoon / evening)
 → objection handling (price, later, not ordered, change)
 → optional upsell / prepaid offer (merchant-flagged only)
 → close + next steps
 → post-call analysis + outcome
```

**Hard limits:**

- Max duration 3 minutes (configurable).
- Max 3 re-prompts per question.
- Max 2 seconds of silence before re-prompt.
- End politely on abuse.
- Never promise a delivery date the merchant has not configured.

### 7.3 Outcome schema

Every call produces a structured record. This is the product's real output.

```json
{
  "outcome": "confirmed",
  "confidence": 0.94,
  "language": "bn",
  "duration_sec": 78,
  "slots": {
    "recipient_name": "Rahim",
    "product": "Cotton Kurti",
    "variant": "M / Blue",
    "quantity": 1,
    "amount_bdt": 1450,
    "address": "House 12, Road 4, Dhanmondi",
    "area": "Dhanmondi, Dhaka",
    "delivery_window": "evening"
  },
  "changes": [
    { "field": "address", "from": "Dhanmondi", "to": "House 12, Road 4, Dhanmondi" }
  ],
  "objections": [],
  "upsell": { "offered": false, "accepted": false, "item": null },
  "consent_recording": true,
  "flags": []
}
```

**Outcome enum:**

| Outcome | Meaning | Merchant action |
|---|---|---|
| `confirmed` | Customer confirmed product, amount, address | Ship |
| `confirmed_with_changes` | Confirmed after address/variant/qty change | Update order, ship |
| `cancelled_by_customer` | Customer cancelled | Do not ship; refund if prepaid |
| `wrong_number` | Number invalid or not the customer | Hold; contact via FB/SMS |
| `no_answer` | No answer after retries | Hold; manual follow-up |
| `callback_requested` | Customer asked for a later call | Schedule callback |
| `suspected_fraud` | Inconsistent answers, known bad number | Manual review |
| `failed` | Technical failure | Retry / escalate |

### 7.4 Bangla-specific design

Bangla voice is the moat and the hardest engineering problem. Treat it as a first-class product concern, not a vendor setting.

**Language behavior:**

- Default to **Bangla (bn-BD)**, with natural code-switching for words like "অর্ডার", "ডেলিভারি", "কনফার্ম", "ক্যাশ অন ডেলিভারি".
- Switch to English if the customer speaks English or asks.
- Do not use West Bengal idioms or overly formal literary Bangla. Use neutral, polite, conversational Dhaka Bangla.
- Say amounts in Bangla words and repeat the number: "মোট এক হাজার চারশো পঞ্চাশ টাকা।"
- For names and addresses, ask the customer to repeat and read it back. ASR will struggle with names; do not pretend it won't.

**Voice and audio:**

- Female voice, 25–40, friendly, neutral accent, slightly faster than formal narration.
- 8 kHz telephony audio; test on feature phones, not just laptop speakers.
- Noise suppression on; expect street/TV/fan noise.
- Target end-to-end latency **< 1.5 seconds**; barge-in must work.
- A/B test voices on **answer rate and confirmation rate**, not on how "natural" they sound to you.
- Consider a licensed Bangla voice actor + a consented voice clone for a distinctive, consistent brand voice. Get written consent and define usage rights.

**Vendor options (validate with real calls):**

| Layer | Options | Notes |
|---|---|---|
| STT | Google Chirp 3 `bn-BD` (preview), Azure Speech, Deepgram (check Bengali), local ASR | Keyword boosting for product names; custom vocabulary |
| LLM | GPT-4.1-mini / Gemini Flash / Claude Haiku class | Structured output; low temperature; small context |
| TTS | ElevenLabs v4/v3 Bengali, Azure `bn-BD` Nabanita/Pradeep, Google Gemini-TTS `bn-BD` | ElevenLabs v3/v4 supports Bengali; Azure has dedicated BD voices |
| Orchestration | Vapi, Retell, LiveKit/pipecat, Plivo Voice AI | Start managed; abstract behind an interface |
| Telephony | Twilio, Plivo, local BTRC-licensed SIP trunk | Local CLI matters for answer rates |

**Do not** use a premium reasoning model for every turn. Use a small, fast model with a tight prompt and a structured output schema; reserve a bigger model for post-call analysis.

### 7.5 Guardrails

- No medical, legal, or financial advice.
- No discounts or promises outside the merchant's configured rules.
- No pressure tactics, no repeated calls after opt-out.
- No discussion of other customers' data.
- No raw PII in logs or third-party analytics.
- Redact sensitive data from transcripts where possible.
- Human review queue for low-confidence or flagged calls.
- Prompt-injection protection: the customer's speech is data, not instructions.

### 7.6 Testing and quality

Before the first merchant pilot:

- 50 scripted scenarios (confirm, cancel, address change, price objection, wrong number, abusive, silence, language switch, voicemail).
- 200 real calls to staff/friends/merchants in Bangla.
- Measure:
  - answer rate,
  - slot accuracy (product, amount, address),
  - confirmation rate,
  - average call duration,
  - human escalation rate,
  - ASR word error rate on a labeled sample.
- Build a **golden set** of 30 recorded calls; run it before every prompt/model change.
- Record every call (with disclosure) and review a sample weekly.

---

## 8. Architecture

### 8.1 High-level

```mermaid
flowchart LR
  subgraph Sources
    W[Website / API / CSV]
    M[Messenger]
    C[Comments]
    L[Lead Ads]
  end

  W --> N[Order Normalizer]
  M --> N
  C --> N
  L --> N

  N --> DB[(Postgres)]
  N --> Q[Call Queue + Rules]

  Q --> V[Voice Agent]
  V --> T[Telephony Provider]
  T --> P[Customer Phone]

  V --> A[Outcome Analyzer]
  A --> DB
  A --> H[Webhooks / Courier]
  DB --> UI[Merchant Dashboard]
  A --> R[(Recordings / Transcripts)]
```

### 8.2 Components

| Component | Responsibility | MVP choice |
|---|---|---|
| **Web app** | Dashboard, onboarding, settings, analytics | Next.js + TypeScript + Tailwind |
| **API** | Orders, calls, webhooks, auth, billing | Next.js route handlers or Fastify |
| **Database** | Tenants, orders, calls, outcomes, usage | Postgres (Neon/Supabase) + Drizzle/Prisma |
| **Queue** | Call scheduling, retries, webhook processing | Redis + BullMQ (Upstash) |
| **Voice provider** | Create/end/transfer calls; webhooks | Vapi first; `MockVoiceProvider` for dev |
| **Telephony** | PSTN connectivity, caller ID | Twilio/Plivo first; local SIP later |
| **Meta adapter** | OAuth, webhooks, send API, window guard | Custom module |
| **Outcome analyzer** | Transcript → structured JSON | LLM with schema validation |
| **Storage** | Recordings, exports | S3-compatible (R2/S3) |
| **Observability** | Errors, call traces, latency | Sentry + OpenTelemetry + call logs |
| **Billing** | Plans, usage, invoices | Manual invoices first; Stripe/Paddle later + bKash/SSLCommerz |

### 8.3 Data model (core tables)

```text
organizations        (id, name, plan, timezone, created_at)
users                (id, org_id, email, role, ...)
channel_connections  (id, org_id, provider, external_id, token_enc, status, scopes)
contacts             (id, org_id, name, phone_enc, email, locale, consent_json)
orders               (id, org_id, external_id, source, status, amount, currency,
                      cod, address_json, items_json, consent_at, created_at)
order_items          (id, order_id, sku, name, variant, qty, unit_price)
calls                (id, order_id, provider, provider_call_id, status,
                      started_at, ended_at, duration_sec, recording_url,
                      transcript_json, outcome, confidence, cost_cents)
call_events          (id, call_id, type, payload_json, created_at)
message_threads      (id, org_id, channel, external_thread_id, psid,
                      window_expires_at, last_user_message_at, opted_out)
messages             (id, thread_id, direction, type, body, meta_json, sent_at)
webhook_events       (id, org_id, source, external_id, payload_json,
                      status, attempts, last_error)
usage_records        (id, org_id, order_id, call_id, unit, qty, cost_cents)
audit_logs           (id, org_id, user_id, action, target, meta_json, created_at)
```

**Rules:**

- Every table has `org_id`; enforce tenant isolation in the query layer or with Postgres RLS.
- Phone numbers and addresses are encrypted at rest.
- Soft-delete + hard-delete job for retention.
- Idempotency keys on order ingest and call creation.

### 8.4 Voice provider abstraction

Keep the voice vendor swappable from day one.

```ts
export interface VoiceProvider {
  createCall(input: {
    to: string;
    from: string;
    assistantId: string;
    metadata: Record<string, string>;
    variableValues: Record<string, string>;
  }): Promise<{ providerCallId: string; status: string }>;

  endCall(providerCallId: string): Promise<void>;
  transferCall(providerCallId: string, to: string): Promise<void>;
  verifyWebhook(headers: Record<string, string>, rawBody: string): boolean;
  parseWebhook(body: unknown): CallEvent[];
}
```

Implement:

- `VapiVoiceProvider` (or Retell) for production,
- `MockVoiceProvider` for local dev and demos,
- `TwilioVoiceProvider` later if you build your own pipeline.

**Why:** Vapi/Retell are fast to start but charge a platform fee and may not support every Bangla voice/STT combination. You will likely move to a custom pipeline at scale. The interface protects you.

### 8.5 Multi-tenancy and security

- Tenant isolation: `org_id` on every row; RLS or a repository guard.
- Auth: email magic link + Google SSO; role-based access (`owner`, `admin`, `agent`, `viewer`).
- Secrets: KMS/envelope encryption for Page tokens, API keys, and phone numbers.
- Webhooks: HMAC signatures, replay protection, idempotency.
- Recording access: signed URLs, short expiry, audit log.
- PII: minimize, encrypt, retention policy, deletion workflow.
- Backups: daily Postgres snapshots; test restore quarterly.
- Rate limits: per tenant and per provider; backoff on Meta/Twilio errors.
- Compliance: privacy policy, DPA, subprocessor list, data deletion request flow.

### 8.6 Why buy the voice stack first

| Approach | Time to first call | Control | Cost at scale | Recommendation |
|---|---|---|---|---|
| Vapi/Retell + Twilio/Plivo | Days | Medium | Higher | **Start here** |
| LiveKit/pipecat + own STT/LLM/TTS | Weeks | High | Lower | Migrate at ~5k+ min/month |
| Full custom SIP/media | Months | Highest | Lowest | Only with a telecom partner |

The first 500 calls are for learning, not for optimizing cost. Buy speed, then own the stack.

---

## 9. Unit economics and pricing

> All numbers are hypotheses as of October 2026. Validate with real calls and real invoices. USD→BDT assumed at ~৳120.

### 9.1 Cost per call

| Component | Managed stack | Local SIP stack (target) |
|---|---:|---:|
| Telephony (BD outbound) | $0.060/min (Twilio) or $0.046/min (Plivo) | ~$0.010/min (local trunk) |
| Voice orchestration | $0.050/min (Vapi) | ~$0.010/min (self-hosted) |
| STT + LLM + TTS | ~$0.040/min | ~$0.030/min |
| **All-in per minute** | **~$0.150 (~৳18)** | **~$0.050 (~৳6)** |
| Blended minutes per attempt (75% answered at 1.3 min + 25% no-answer at 0.3 min) | 1.05 min | 1.05 min |
| Cost per attempt | ~$0.16 (~৳19) | ~$0.05 (~৳6) |
| With 30% retry overhead | ~$0.21 (~৳25) | ~$0.07 (~৳8) |

**Reality check:** managed telephony + managed voice is **2–3× more expensive** than a local stack. At ৳15/call, the managed stack loses money. Either price higher, target higher-AOV merchants, or move to local SIP before scaling.

### 9.2 Pricing options

| Model | Example | Pros | Cons |
|---|---|---|---|
| **Per call** | ৳18–25 per attempted call | Simple, predictable | Merchant pays for no-answers; feels like a cost |
| **Per confirmed order** | ৳30–40 per confirmed order | Aligns with value; easy to sell | You eat the cost of cancellations/no-answers |
| **Platform + usage** | ৳2,500/mo + ৳12/call | Recurring revenue; covers fixed costs | Higher friction for small sellers |
| **Outcome share** | 10–15% of saved RTO | Strong alignment | Hard to measure; disputed attribution |

**Recommended for the pilot:** platform fee + per-call usage, with a free 100-call trial. Move to per-confirmed-order once you have proof and local telephony.

### 9.3 Worked example: 1,000 orders/month

Assumptions: AOV ৳1,500; COD 100%; RTO 30%; cost per RTO ৳180; confirmation rate 70%; call cost ৳10 (local stack); price ৳18/call.

| Line | Value |
|---|---:|
| Orders/month | 1,000 |
| Calls (incl. retries) | 1,200 |
| Call cost | ৳12,000 |
| Revenue at ৳18/call | ৳21,600 |
| Gross profit | ৳9,600 (44%) |
| Fixed infra/support (allocated) | ৳4,000 |
| Contribution | ৳5,600 |

Now the merchant's value:

| Line | Value |
|---|---:|
| RTO before | 300 returns × ৳180 = ৳54,000 |
| RTO after an 8-point improvement | 220 returns × ৳180 = ৳39,600 |
| Courier savings | ৳14,400 |
| Prepaid conversion: 5% × 1,000 × ৳1,500 × 25% margin | ৳18,750 |
| Half a caller's salary avoided | ৳10,000 |
| **Total value** | **~৳43,150** |
| Our price | ৳21,600 |
| **Merchant ROI** | **~2×** |

The ROI is real but not enormous. Two implications:

1. **Do not sell on courier savings alone.** Sell the bundle: confirmation + address repair + prepaid conversion + labor + data.
2. **Target merchants where the math is better:** AOV > ৳1,500, 1,000+ orders/month, high RTO, weak existing process.

### 9.4 The uncomfortable truth about Bangladeshi labor

A human caller in Dhaka costs roughly ৳18,000–25,000/month and can make 150–250 calls/day. That is **৳3–5 per call**. An AI call at ৳10–26 is more expensive per call.

So the AI cannot win on **cost per call**. It wins on:

- **speed** (5 minutes vs next day),
- **availability** (24/7, peaks, Eid),
- **consistency** (same script, every order),
- **data** (structured outcomes, recordings, reasons),
- **scale without hiring**,
- **address repair and upsell** that a bored caller skips,
- **integration** with orders, couriers, and analytics.

Position accordingly. If you sell "cheaper calls," you will lose to a ৳15,000/month agent.

### 9.5 Telephony and caller ID in Bangladesh

This is an operational risk, not just a line item.

- Twilio lists BD outbound at **$0.060/min**; Plivo at **$0.046/min**. Plivo does not support BD inbound numbers.
- International caller IDs often get **lower answer rates**. A customer who sees a foreign number may not pick up.
- For production, you likely need a **local SIP trunk / CLI from a BTRC-licensed provider**, or a partnership with a local call-center platform.
- Validate: answer rate by CLI type, call quality, DTMF reliability, latency, and cost.
- Get local legal advice on unsolicited calling, recording consent, and data protection.

**Do not sign a merchant contract promising "95% answer rate" before you have tested local CLI.**

---

## 10. Go-to-market

### 10.1 The first 10 customers

Do not launch broadly. Find 10 sellers who feel the pain weekly.

**Sourcing:**

- Facebook seller groups and e-commerce communities (post value, not spam).
- Instagram/Facebook pages in fashion, cosmetics, home, gadgets with high comment volume.
- Courier partner account managers (Pathao/Steadfast/RedX/Paperfly) — ask for introductions to merchants with high RTO.
- Facebook ads agencies managing multiple Pages.
- Local e-commerce associations and startup communities.

**Outreach:**

> "We call every COD order within 5 minutes in Bangla and confirm the address before you ship. We cut returns by 8–15 points in pilots. Want a free 100-call test on your next 100 orders?"

**The demo that sells:** ask for their phone number, trigger a live AI call in Bangla, and let them experience it. Then show the dashboard with the transcript and outcome. This is more persuasive than any slide.

### 10.2 The pilot

- 2 weeks.
- 50/50 split of new COD orders (alternate by order ID) to create a control group.
- Success criteria agreed in writing:
  - answer rate > 55%,
  - confirmation rate > 65%,
  - RTO reduction vs control > 5 points (directional in 2 weeks; keep measuring),
  - merchant would pay for it.
- Weekly review: listen to 5 calls together, fix the script, fix the data.
- No long contract. Convert to paid only if the numbers are real.

### 10.3 Pricing and packaging

Start simple:

| Plan | Price | Includes | Target |
|---|---:|---|---|
| Pilot | Free | 100 calls, 1 Page, 1 user | First 5 merchants |
| Starter | ৳2,500/mo | 150 calls, 1 Page, 2 users | 100–300 orders/mo |
| Growth | ৳6,000/mo | 500 calls, 3 Pages, analytics | 500–1,000 orders/mo |
| Scale | ৳12,000/mo | 1,200 calls, API, priority support | 1,000+ orders/mo |
| Overage | ৳12–15/call | | |

**Do not offer unlimited calls.** Voice costs are real; unlimited plans attract the worst usage and destroy margin.

### 10.4 Channels and partnerships

- **Agencies:** white-label the dashboard; they sell to their clients; you charge per call. Faster distribution than direct sales.
- **Couriers/3PLs:** bundle confirmation into their merchant onboarding; they already own the RTO problem.
- **Call centers:** offer AI as an overflow/after-hours layer, not a replacement. They have the merchant relationships.
- **Shopify/WooCommerce agencies:** integration partners for website merchants.
- **Content:** an RTO calculator, a Bangla guide to COD confirmation, and case studies with real numbers.

### 10.5 Sales objections and answers

| Objection | Answer |
|---|---|
| "AI can't speak Bangla properly." | Live demo; recordings; slot accuracy; human fallback. Let the call speak for itself. |
| "Customers will hang up." | Pre-call Messenger/SMS, local CLI, short script; show answer-rate data from the pilot. |
| "We already have callers." | AI handles 24/7, peaks, and repeat attempts; humans handle exceptions and high-value orders. |
| "Is this legal?" | Consent at checkout/chat, AI disclosure, recording disclosure, DNC list, local counsel. |
| "Will Facebook ban us?" | We follow the 24h/7d windows, use utility templates only for updates, and never cold-DM. |
| "Too expensive." | Per-confirmed-order pricing; ROI calculator; free pilot; compare to one prevented RTO. |
| "We tried IVR." | IVR is keypad-based and hated; this is natural conversation with address repair. |

### 10.6 What not to do

- Do not promise "unlimited Facebook messages."
- Do not scrape Facebook profiles or buy phone numbers.
- Do not call people who never ordered or opted in.
- Do not use the HUMAN_AGENT tag for bot messages.
- Do not launch with 10 features; launch with one workflow.

---

## 11. Roadmap

### Phase 0 — Validation (week 0, ~1 week)

- Interview 10 merchants; listen to 20 real confirmation calls.
- Collect 100 real order records (anonymized) to understand fields and failure modes.
- Test Bangla STT/TTS with 3 vendors on real phone audio.
- Test Twilio/Plivo BD outbound and a local CLI option.
- Write the v1 script and outcome schema.
- Start Meta Business Verification.
- **Exit criteria:** a call recording that a merchant says "yes, I would use that."

### Phase 1 — Core engine (weeks 1–3)

- Auth, orgs, orders, calls, outcomes.
- CSV import + REST API + idempotency.
- Mock voice provider + simulator.
- Vapi/Retell adapter + Bangla agent.
- Retry/scheduling/calling hours/DNC.
- Dashboard: order list, call detail, transcript, recording, outcome.
- Test-call button.
- **Exit criteria:** 100 real calls to staff/friendly merchants; 90%+ slot accuracy on a labeled sample.

### Phase 2 — Pilot (weeks 4–6)

- 3–5 merchants; 1,000+ calls.
- A/B control group; RTO measurement.
- WooCommerce plugin.
- SMS pre-notice + missed-call follow-up.
- Billing/usage metering (manual invoices okay).
- Script iteration from real calls.
- **Exit criteria:** 3 merchants willing to pay; confirmation rate > 65%; cost per call < ৳15.

### Phase 3 — Facebook (weeks 7–10; review starts week 2)

- Meta app + webhook receiver + event log.
- Page connect OAuth + token vault.
- Messenger inbox: receive, reply, extract order, create order.
- Confirmation card + quick replies.
- Comment-to-DM (one private reply).
- Lead Ads fast-call.
- Utility template for order status (if approved).
- **Exit criteria:** App Review approved; 3 merchants live on Messenger; no policy warnings.

### Phase 4 — Scale (weeks 11–16)

- Local SIP/CLI migration.
- Courier integrations (Pathao/Steadfast/RedX/Paperfly).
- Agency white-label.
- Prepaid conversion + upsell.
- Analytics: RTO, saved revenue, agent performance.
- Security review, DPA, SOC2-lite readiness.
- **Exit criteria:** 20–30 paying merchants; positive contribution margin; < 5% monthly churn.

### Team (lean)

| Role | Focus |
|---|---|
| Founder / product | Merchant interviews, sales, script, pricing |
| Full-stack engineer | Dashboard, API, Meta adapter, billing |
| AI/voice engineer (contract or co-founder) | STT/TTS/LLM pipeline, latency, Bangla quality |
| Bangla QA / ops (part-time) | Listen to calls, label data, tune scripts |

A solo founder can do Phase 0–1 with a voice vendor, but Bangla QA is not optional.

---

## 12. Metrics and proof

### 12.1 North-star metric

**Confirmed orders per month** (and **RTO avoided**). Not calls, not minutes.

### 12.2 Funnel

| Stage | Metric | Target |
|---|---|---:|
| Order ingested | Orders/day | — |
| Call attempted | Attempt rate | > 98% |
| Answered | Answer rate | > 55% |
| Meaningful conversation | > 20 seconds | > 80% of answers |
| Confirmed | Confirmation rate | > 65% |
| Address repaired | Change rate | 10–20% |
| Delivered | Delivery rate | Track vs control |
| Returned | RTO rate | Down 5–15 points |

### 12.3 Quality

- Slot accuracy (product, variant, qty, amount, address): > 95% on labeled sample.
- ASR word error rate: track weekly; improve with vocabulary.
- Human escalation rate: < 15%.
- Complaint rate: < 1%.
- Average call duration: 60–120s.
- Technical failure rate: < 2%.

### 12.4 Business

- Activation: merchant sends first order within 48 hours.
- Weekly active merchants: > 70% of paying.
- Gross margin per call: > 40%.
- CAC payback: < 3 months.
- Logo churn: < 5%/month.
- Net revenue retention: > 100% (usage growth).

### 12.5 How to prove RTO impact

- Randomize by order ID: 50% AI confirmation, 50% existing process.
- Run for at least 2 weeks (ideally 4).
- Compare RTO, delivery rate, cancellation rate, and prepaid conversion.
- Control for category, AOV, day of week, and courier.
- Publish a case study with the merchant's permission.

**Do not claim "we cut RTO by 30%" without a control group.** Merchants will ask, and the number will be challenged.

---

## 13. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Meta App Review delay or rejection | High | High | Start week 1; minimal permissions; screencast; website/CSV fallback |
| Policy violation (window/tag misuse) | Medium | High | Central window guard; no cold DMs; HUMAN_AGENT only for humans; legal review |
| Bangla ASR accuracy | High | Medium | Real-call data; custom vocabulary; DTMF fallback; human handoff |
| Low answer rates / foreign CLI | High | High | Local SIP/CLI; pre-call notice; timing; retries; A/B test |
| Telephony regulation / consent | Medium | High | BTRC-licensed partner; consent records; DNC; local counsel |
| Unit economics worse than planned | Medium | High | Local stack; short calls; higher-AOV ICP; per-confirmed pricing |
| Merchant trust in AI | Medium | Medium | Pilot; recordings; transparency; money-back guarantee |
| Competition from call centers/IVR | High | Medium | Speed, data, Bangla, FB-native intake, integrations |
| Data privacy incident | Low | High | Encryption; retention; access control; deletion; incident plan |
| Key-person dependency | Low | Medium | Document scripts/prompts; contractor bench |

### Competition map

| Competitor type | Strength | Weakness | Our edge |
|---|---|---|---|
| Human call centers | Cheap per call, trusted, flexible | Slow, inconsistent, no data, hard to scale at peaks | 24/7, speed, structured outcomes |
| IVR providers | Cheap, simple | Keypad menus, high hang-up, no address repair | Natural Bangla conversation |
| WhatsApp/Messenger automation tools | Cheap chat, familiar | No voice, policy-limited, no telephony | Voice + FB intake + courier workflow |
| Horizontal AI voice platforms (Vapi, Retell, Bland) | Powerful, flexible | No FB workflow, no Bangla tuning, no local ops | Vertical product, Bangla, integrations |
| Courier confirmation services | Bundled, cheap | Generic, not merchant-branded, limited data | Merchant-owned workflow + analytics |

---

## 14. Open decisions

These are the decisions that change the plan. My recommendation is in bold.

### 14.1 Which market first?

- **A. Bangladesh Facebook/COD sellers (recommended).** You have language and distribution advantage; the pain is acute; the unit economics are tight but workable with local SIP.
- B. Global English e-commerce (Shopify/Woo). Higher prices, easier telephony, but you compete with many post-purchase tools and lose your language moat.
- C. B2B distributor/dealer order confirmation. Larger order values, fewer calls, but different consent and integration model.
- D. Gulf/Southeast Asia South Asian diaspora sellers. Higher AOV and calling costs, but harder distribution.

**Pick A for the first 90 days. Expand to B/C only after the BD workflow is proven.**

### 14.2 Voice-first or chat-first?

- **Voice-first (recommended for the wedge).** The phone call verifies reachability and intent; it is the product's differentiator.
- Chat-first is cheaper and fits Messenger, but customers ignore chat and it does not verify the phone number.
- Best long-term: **chat first when the window is open, voice when it is closed or when the order is high-value.**

### 14.3 Build or buy the voice stack?

- **Buy for the first 1,000 calls** (Vapi/Retell + Twilio/Plivo). Learn what "good" sounds like.
- **Build/self-host once you exceed ~5,000 minutes/month** or need local SIP/custom Bangla models.
- Keep the `VoiceProvider` interface from day one.

### 14.4 Direct or white-label?

- **Direct for the first 10 merchants** to learn the workflow and script.
- **White-label to agencies/couriers after** you have a repeatable playbook and case studies.
- White-label is the faster path to scale in BD, but it hides the merchant relationship and compresses margin.

### 14.5 Pricing model?

- **Pilot:** free 100 calls.
- **Paid:** platform fee + per-call, then per-confirmed-order once you have proof.
- Avoid unlimited plans and pure outcome-share until attribution is reliable.

### 14.6 What exactly does "confirm the purchase" mean?

- Consumer COD order (assumed): confirm intent, address, delivery window.
- B2B purchase order: confirm quantity, SKU, price, delivery date, credit terms.
- Appointment/booking: confirm time, location, reschedule.
- The engine is the same; the script, slots, and integrations differ. **Pick one for v1.**

---

## 15. What to build first in this repo

If we start coding now, this is the order I would use. Each step is shippable and demoable.

### Step 1 — Foundation (2–3 days)

- Next.js + TypeScript + Tailwind.
- Auth + organizations + users.
- Postgres schema: orgs, orders, calls, outcomes, usage.
- Seed data + local dev script.

### Step 2 — Order intake (2–3 days)

- `POST /api/v1/orders` with idempotency key.
- CSV upload.
- Order list + detail.
- Consent capture fields.
- **Demo:** upload 20 orders, see them in the dashboard.

### Step 3 — Mock call engine (3–4 days)

- `MockVoiceProvider` + a browser "call simulator."
- Script/state machine with slots.
- Outcome JSON + confidence.
- Call detail page: transcript, outcome, changes.
- **Demo:** simulate a Bangla/English confirmation call without spending on telephony.

### Step 4 — Real voice (3–5 days)

- Vapi/Retell adapter.
- Bangla agent config (prompt, voice, slots, guardrails).
- Test-call button.
- Webhook for call status/transcript.
- Recording playback.
- **Demo:** call your own phone from the dashboard.

### Step 5 — Rules and reliability (3–4 days)

- Call window, retries, DNC, max attempts.
- Queue + worker + idempotency.
- SMS pre-notice (local aggregator or Twilio).
- Failure handling and alerts.

### Step 6 — Meta foundation (3–5 days, can run in parallel)

- Webhook verification + signature check.
- Event log + simulator.
- Page connect OAuth + encrypted token storage.
- Send API wrapper + window guard.
- **Note:** you can test with your own Page in development mode before App Review.

### Step 7 — Messenger workflow (5–7 days, after App Review or with dev-mode Page)

- Receive messages.
- Extract order with an LLM + schema.
- Create order + confirmation card.
- Trigger call.
- Human handoff.

### Step 8 — Pilot hardening (1–2 weeks)

- Usage metering + invoices.
- Analytics: answer/confirmation/RTO.
- Export + webhook.
- Audit logs + retention job.
- Onboarding checklist.

**Do not build in v1:** inbound support, multi-language beyond BN/EN, Shopify App Store listing, mobile app, custom LLM training, full CRM.

---

## 16. Sources

Primary sources for the policy and pricing claims in this document (accessed 23 October 2026):

- Meta, **Messenger Platform and IG Messaging API policy** — standard 24h/7d window, private replies, responsiveness: https://developers.facebook.com/documentation/business-messaging/messenger-platform/policy
- Meta, **Messenger Platform changelog** — 28 Sep 2026 7-day Click-to-Messenger window; 27 Apr 2026 deprecated tags: https://developers.facebook.com/documentation/business-messaging/messenger-platform/changelog
- Meta, **Send API reference** — `messaging_type`, `HUMAN_AGENT`, tags: https://developers.facebook.com/docs/messenger-platform/reference/send-api/
- Meta, **Send a utility message** — `UTILITY`, `page_utility_messaging`, templates: https://developers.facebook.com/documentation/business-messaging/messenger-platform/send-messages/utility-messages
- Meta, **Lead Ads** — App Review, `leads_retrieval`, 90-day retention: https://developers.facebook.com/documentation/ads-commerce/marketing-api/guides/lead-ads
- Meta, **Lead Ads webhooks** — required permissions and setup: https://developers.facebook.com/docs/graph-api/webhooks/getting-started/webhooks-for-leadgen/
- Meta, **Marketing Messages on Messenger** — paid opt-in outside the window: https://developers.facebook.com/documentation/business-messaging/messenger-platform/marketing-messages-on-messenger
- Twilio, **Voice pricing — Bangladesh** ($0.060/min): https://www.twilio.com/en-us/voice/pricing/bd
- Plivo, **Voice pricing — Bangladesh** ($0.046/min): https://www.plivo.com/voice/pricing/bd/
- ElevenLabs, **Models and languages** (Bengali on v3/v4): https://elevenlabs.io/docs/overview/models
- Microsoft Azure, **Speech language support** (`bn-BD` voices; STT availability): https://learn.microsoft.com/en-us/azure/ai-services/speech-service/language-support
- Google Cloud, **Chirp 3 transcription** (`bn-BD` preview): https://docs.cloud.google.com/speech-to-text/docs/models/chirp-3
- Google Cloud, **Gemini-TTS** (`bn-BD` GA): https://docs.cloud.google.com/text-to-speech/docs/gemini-tts
- RTO benchmarks (secondary, for directional context): industry reports cited in §2.1; validate with local merchant data.

---

## Appendix A — Bangla call script

> Keep each turn short. Pause. Let the customer speak. Read back critical data.

### A.1 Opening

**Agent:**
"আসসালামু আলাইকুম। আমি {দোকানের নাম} থেকে বলছি। আপনার অর্ডারটি কনফার্ম করার জন্য কল দিয়েছি। আমি {দোকানের নাম}-এর একটি অটোমেটেড অ্যাসিস্ট্যান্ট। এক মিনিট কথা বলতে পারি?"

*(Peace be upon you. I'm calling from {shop}. I'm calling to confirm your order. I'm an automated assistant from {shop}. Can you talk for a minute?)*

If the customer says no:
"ঠিক আছে, কখন কল দিলে সুবিধা হবে?"

### A.2 Order confirmation

**Agent:**
"আপনি {প্রোডাক্টের নাম}, {সাইজ/কালার}, {পরিমাণ}টি — মোট {টাকা} টাকা ক্যাশ অন ডেলিভারিতে অর্ডার করেছেন। ঠিক আছে?"

*(You ordered {product}, {size/color}, {quantity} — total {amount} taka cash on delivery. Is that correct?)*

**If yes:** move to address.
**If no:** "কী পরিবর্তন করতে চান — প্রোডাক্ট, সাইজ, নাকি পরিমাণ?"

### A.3 Address confirmation

**Agent:**
"ডেলিভারির ঠিকানা: {এলাকা}, {বাসা/রোড}। ঠিক আছে?"

**If no:**
"নতুন ঠিকানাটি বলুন — এলাকা, বাসা/রোড, এবং কাছের কোনো পরিচিত জায়গা। আমি পড়ে শুনাচ্ছি।"

*(Tell me the new address — area, house/road, and a nearby landmark. I'll read it back.)*

### A.4 Delivery preference

**Agent:**
"ডেলিভারি কখন নিতে চান — সকালে, দুপুরে, নাকি বিকেলে?"

### A.5 Close

**Agent:**
"ধন্যবাদ। আপনার অর্ডার কনফার্ম হয়েছে। ডেলিভারির আগে কুরিয়ার আপনাকে কল দেবে। ভালো থাকবেন।"

### A.6 Objection branches

| Customer says | Agent responds |
|---|---|
| "দাম বেশি" (too expensive) | "দামটি অর্ডারের সময়েই নির্ধারিত ছিল। আপনি চাইলে অর্ডারটি বাতিল করতে পারেন, অথবা ডেলিভারির সময় দেখে নিতে পারেন।" |
| "পরে নিব" (I'll take it later) | "ঠিক আছে, কখন কল দিলে সুবিধা হবে — আজ বিকেলে, নাকি কাল সকালে?" |
| "আমি অর্ডার করিনি" (I didn't order) | "দুঃখিত, হয়তো ভুল হয়েছে। অর্ডারটি বাতিল করে দিচ্ছি। আপনার সময়ের জন্য দুঃখিত।" |
| "ভুল নম্বর" (wrong number) | "দুঃখিত, বিরক্তির জন্য ক্ষমা করবেন। আমি নম্বরটি বাদ দিয়ে দিচ্ছি।" |
| "ঠিকানা বদলাতে চাই" | "অবশ্যই। নতুন ঠিকানাটি বলুন, আমি পড়ে শুনাচ্ছি।" |
| "মানুষের সাথে কথা বলতে চাই" | "অবশ্যই, আপনার কলটি {দোকানের} একজন প্রতিনিধির কাছে দিচ্ছি। একটু অপেক্ষা করুন।" |

### A.7 DTMF fallback

"বোতাম চাপতে চাইলে — কনফার্ম করতে ১ চাপুন, বাতিল করতে ২ চাপুন।"

### A.8 Voicemail

"আসসালামু আলাইকুম, {দোকানের নাম} থেকে বলছি। আপনার অর্ডারটি কনফার্ম করতে {নম্বর} নম্বরে কল করুন। ধন্যবাদ।"

Then send an SMS with the order summary and a callback number.

### A.9 English fallback

"Would you prefer to continue in English?"
"Hi, I'm calling from {shop} to confirm your order. Is now a good time?"

---

## Appendix B — Facebook App Review checklist

### B.1 Before submission

- [ ] Meta Business Portfolio created and **Business Verification** submitted.
- [ ] App type: Business.
- [ ] Privacy policy URL live and covers: Page data, Messenger messages, comments, Lead Ads, retention, deletion, sharing, contact.
- [ ] Terms of service URL live.
- [ ] Data deletion callback implemented and tested; instructions URL live.
- [ ] Deauthorize callback implemented.
- [ ] HTTPS webhook with valid certificate; responds < 5s; signature verified.
- [ ] Test Page with real content and a real conversation.
- [ ] Test lead form with a test lead (if applying for `leads_retrieval`).
- [ ] Utility template created and sent (if applying for `page_utility_messaging`).
- [ ] Reviewer test account with the right Page role.
- [ ] No fake data, no placeholder screens, no lorem ipsum.

### B.2 Screencast outline (3–5 minutes, narrated)

1. **Intro (15s):** who you are, what the app does.
2. **Login (20s):** Facebook Login, select Page, grant permissions.
3. **Messenger (60s):** send a message from a test user; show it in the dashboard; reply; create an order.
4. **Voice call (45s):** trigger the call; show the call record/outcome (or a recording).
5. **Lead Ads (40s):** submit a test lead; show the webhook and the lead in the app (if applicable).
6. **Utility template (30s):** show the approved template and a send (if applicable).
7. **Data deletion (20s):** show the deletion request and confirmation.
8. **Close (10s):** what permissions you need and why.

### B.3 Permission justifications (templates)

- `pages_show_list`: "To let a merchant choose which Facebook Page to connect to their account."
- `pages_manage_metadata`: "To subscribe our webhook to the Page's messages and lead events."
- `pages_messaging`: "To read and reply to customer messages about orders, and send order confirmation updates."
- `pages_read_engagement`: "To read comments on the Page's posts and send one private reply when a customer asks about an order."
- `page_utility_messaging`: "To send approved non-promotional order status updates outside the 24-hour window."
- `leads_retrieval`: "To retrieve Lead Ads form submissions so the merchant can call the customer and confirm the order."

### B.4 Common rejection reasons

- Webhook times out or fails verification.
- Screencast shows settings, not a real end-to-end flow.
- Permission requested but not used in the demo.
- Privacy policy missing Messenger/lead data or deletion.
- Test Page not connected to the app.
- Fake data.
- Requesting too many permissions at once.

---

## Appendix C — API payloads

### C.0 Consent text (use at checkout and in chat)

**Website checkout (English):**
> By placing this order, I agree to be contacted by phone about this order. Calls may be recorded for quality and training.

**Website checkout (Bangla):**
> অর্ডারটি নিশ্চিত করার জন্য আমি ফোনে যোগাযোগ করতে সম্মতি দিচ্ছি। মান উন্নয়নের জন্য কল রেকর্ড করা হতে পারে।

**Messenger chat (before asking for the phone number):**
> আপনার নম্বরটি শুধু এই অর্ডারটি কনফার্ম করার জন্য ব্যবহার করা হবে। ঠিক আছে?

Store the consent text version, timestamp, source, and IP/session where applicable. Consent must be affirmative (a checked box), not buried in terms.

### C.1 Create order

```http
POST /api/v1/orders
Authorization: Bearer sk_live_...
Idempotency-Key: order-8821
Content-Type: application/json
```

```json
{
  "external_id": "WEB-8821",
  "source": "website",
  "customer": {
    "name": "Rahim Uddin",
    "phone": "+8801712345678",
    "locale": "bn-BD"
  },
  "items": [
    {
      "sku": "KURTI-001",
      "name": "Cotton Kurti",
      "variant": "M / Blue",
      "quantity": 1,
      "unit_price_bdt": 1450
    }
  ],
  "amount_bdt": 1450,
  "payment_method": "cod",
  "address": {
    "line1": "House 12, Road 4",
    "area": "Dhanmondi",
    "city": "Dhaka",
    "postcode": "1209",
    "landmark": "Near Dhanmondi Lake"
  },
  "consent": {
    "call": true,
    "recording": true,
    "source": "checkout_checkbox",
    "captured_at": "2026-10-23T10:15:00Z",
    "text_version": "v1"
  },
  "metadata": {
    "campaign": "eid-2026",
    "page_id": "1234567890"
  }
}
```

### C.2 Order response

```json
{
  "id": "ord_01J...",
  "status": "queued_for_call",
  "call_eta_seconds": 120,
  "outcome": null
}
```

### C.3 Outcome webhook

```http
POST https://merchant.example.com/webhooks/confirm
X-Signature: sha256=...
```

```json
{
  "event": "call.completed",
  "order_id": "ord_01J...",
  "external_id": "WEB-8821",
  "outcome": "confirmed_with_changes",
  "confidence": 0.93,
  "changes": [
    { "field": "address.line1", "from": "House 12", "to": "House 12, Road 4, Flat 3B" }
  ],
  "delivery_window": "evening",
  "recording_url": "https://.../signed",
  "transcript_url": "https://.../signed",
  "completed_at": "2026-10-23T10:22:31Z"
}
```

### C.4 Meta webhook (simplified)

```json
{
  "object": "page",
  "entry": [
    {
      "id": "PAGE_ID",
      "time": 1761210000000,
      "messaging": [
        {
          "sender": { "id": "PSID" },
          "recipient": { "id": "PAGE_ID" },
          "timestamp": 1761210000000,
          "message": { "mid": "m_...", "text": "দাম কত?" }
        }
      ]
    }
  ]
}
```

---

## Appendix D — 30-day validation plan

### Week 1 — Listen and test

- 10 merchant interviews (record with permission).
- Listen to 20 real confirmation calls.
- Collect 100 anonymized orders.
- Test 3 Bangla STT/TTS combinations on real phone audio.
- Test Twilio/Plivo outbound + one local CLI option.
- Draft the script and outcome schema.
- Start Meta Business Verification.

**Deliverable:** a call recording + a one-page script that 3 merchants approve.

### Week 2 — Build the mock + make 100 calls

- Order API + CSV + dashboard.
- Mock call simulator.
- Vapi/Retell integration.
- Call 50 staff/friends + 50 friendly merchant orders.
- Measure answer rate, slot accuracy, duration, failures.

**Deliverable:** 100 real calls and a quality report.

### Week 3 — Pilot with 2 merchants

- 50/50 split.
- Daily call review.
- Fix script/prompt/data issues.
- SMS pre-notice.
- Compare confirmation and cancellation rates.

**Deliverable:** 300–500 calls, 2 merchant testimonials or rejections.

### Week 4 — Decide

Go/no-go questions:

1. Did 5 of 10 merchants say they would pay ৳15–25 per call or ৳30 per confirmed order?
2. Is the answer rate > 50% with the chosen CLI?
3. Is the confirmation rate > 65%?
4. Is the cost per call < ৳15 (or < ৳25 with managed stack)?
5. Did RTO improve directionally vs control?
6. Is Meta review progressing?

**If yes:** raise/build for Phase 2 and submit App Review.
**If no:** narrow the ICP (higher AOV, higher RTO), switch to chat-first, or pivot to white-label for agencies.

---

## The one-page version

- **Product:** AI calls COD buyers in Bangla within 5 minutes, confirms product/price/address, returns a structured outcome.
- **Wedge:** Facebook-first sellers in Bangladesh with 300–3,000 orders/month and painful RTO.
- **Facebook reality:** no outbound calling API; Messenger is intake + in-window chat; phone number must be collected; App Review is the long pole.
- **Build order:** voice engine + website/CSV → pilot → Messenger/Lead Ads → scale.
- **Moat:** Bangla voice quality + FB-native order extraction + courier/outcome data + local support.
- **Business model:** platform fee + per-call, then per-confirmed-order; not unlimited.
- **Biggest risk:** unit economics versus cheap human callers. Local SIP + short calls + higher-AOV ICP are mandatory.
- **Proof:** 50/50 pilot, measured RTO, 3 paying merchants before building the Messenger bot.

---

*This is a living document. Update it after every 100 calls and every merchant interview.*
