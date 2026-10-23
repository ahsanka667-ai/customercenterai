# CustomerCenter AI

**AI order confirmation for Facebook-first B2B sellers — in the seller's language, under the seller's brand.**

**Read the plan: [PLAN.md](./PLAN.md)** (v0.3)

The short version:

- **B2B SaaS:** you sell to sellers; the AI calls their buyers. The seller configures language, voice, brand name, script, hours, and confirmation mode.
- **Every order gets a call by default.** Cost is reduced by owning the voice engine and telephony route — not by skipping calls.
- **Two services:** `apps/web` (dashboard, API, integrations) and `services/voice-engine` (Pipecat/LiveKit + model adapters + SIP/media).
- **Integration hub:** Postgres outbox → event bus → connectors (Facebook, website, WooCommerce, Sheets, couriers, CRMs) → signed webhooks, with retries, mappings, and a dead-letter queue.
- **Facebook is not a phone channel.** Messenger is intake + in-window chat; the call runs over telephony. No outbound calling API, no phone number in the profile, App Review is the long pole.
- **Language is a product setting:** Bangla + English first; new markets are language packs + local telephony, not rewrites.
- **Cost path:** managed stack (pilot) → own orchestration + local SIP + cloud models (early) → self-hosted models at the volume gate.
- **Start Meta Business Verification and App Review in week 1** while building the voice engine and integration hub.

Status: planning / pre-MVP.
