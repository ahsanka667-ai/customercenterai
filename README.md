# CustomerCenter AI

**AI order confirmation for Facebook-first B2B sellers — in the seller's language, under the seller's brand.**

**Read the plan: [PLAN.md](./PLAN.md)** (v0.5)

The short version:

- **B2B SaaS:** you sell to sellers; the AI calls their buyers. The seller configures language, voice, brand name, script, hours, and confirmation mode.
- **Every order gets a call by default.** Cost is reduced by owning the engine and the phone line — not by skipping calls.
- **No GPU.** STT, LLM, and TTS are API calls. A small CPU VPS runs the call loop.
- **No caller API.** No Vapi/Retell/Bland per-minute platform. You own the orchestration: `services/voice-engine` (Pipecat).
- **Own telephony, legally:** run FreeSWITCH/Kamailio yourself in Dhaka, buy the SIP trunk from a BTRC-licensed IPTSP. You do **not** need a telecom licence to use a trunk; you do need one to resell voice.
- **Cost:** BD IPTSP trunks publish ৳0.30–0.40/min + VAT versus Twilio's $0.060/min — roughly 15–18× cheaper on the phone leg. In production, AI APIs become the largest cost, not telephony.
- **Integration hub:** Postgres outbox → event bus → connectors (Facebook, website, WooCommerce, Sheets, couriers, CRMs) → signed webhooks, with retries, mappings, and a dead-letter queue.
- **Facebook is not a phone channel.** Messenger is intake + in-window chat; the call runs over your own telephony. App Review is the long pole.
- **Language is a product setting:** Bangla + English first; new markets are language packs + local telephony, not rewrites.

Status: planning / pre-MVP.
