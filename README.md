# CustomerCenter AI

**AI order confirmation for Facebook-first B2B sellers — in the seller's language, under the seller's brand.**

**Read the plan: [PLAN.md](./PLAN.md)** (v0.4)

The short version:

- **B2B SaaS:** you sell to sellers; the AI calls their buyers. The seller configures language, voice, brand name, script, hours, and confirmation mode.
- **Every order gets a call by default.** Cost is reduced by owning the engine and the telephony route — not by skipping calls.
- **No GPU.** STT, LLM, and TTS are API calls. A small CPU VPS runs the call loop.
- **No caller API.** No Vapi/Retell/Bland per-minute platform. You own the orchestration: `services/voice-engine` (Pipecat) speaks WebSocket media streams directly.
- **Telephony is still required.** Twilio/Plivo media streams first, then a local SIP trunk for cost and caller ID.
- **Integration hub:** Postgres outbox → event bus → connectors (Facebook, website, WooCommerce, Sheets, couriers, CRMs) → signed webhooks, with retries, mappings, and a dead-letter queue.
- **Facebook is not a phone channel.** Messenger is intake + in-window chat; the call runs over telephony.
- **Language is a product setting:** Bangla + English first; new markets are language packs + local telephony, not rewrites.
- **Margin:** ~30–45% on Twilio BD at $0.06/min; ~70% after moving to local SIP. That move is a requirement, not an optimization.

Status: planning / pre-MVP.
