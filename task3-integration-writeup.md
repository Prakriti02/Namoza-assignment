# Task 03 — Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

## Architecture

I'd avoid a native HubSpot form embed and avoid Zapier/Make for the core path — both add either a 3–5 second polling delay or a black-box layer we can't easily debug when the WhatsApp SLA is only 2 minutes. Instead: **the landing page form posts directly to a small serverless function (e.g., a Vercel/Cloudflare Worker or AWS Lambda endpoint), which fans out to three destinations in parallel.**

Flow: Form submit → `POST /api/lead` → the function does three things concurrently rather than sequentially, since none of the three depends on the others succeeding:

1. **HubSpot**: call the HubSpot **Contacts API** directly (`PATCH /crm/v3/objects/contacts` with an `idProperty` lookup), not the Forms API — the Forms API is built for native embeds and CRM-attributed form submissions, not for a custom-built page. A direct API call also gives us control over the dedup key (see below), which the Forms API doesn't.
2. **WhatsApp**: call Namoza's Karix WhatsApp Business API endpoint directly with a pre-approved template message ("Hi {name}, we've received your enquiry for {clinic}...").
3. **Google Ads**: fire the `consultation_form_submitted` conversion via **Google Ads Enhanced Conversions (server-side)**, so it isn't dependent on the browser dataLayer/GTM tag actually completing before the user closes the tab.

A serverless function also lets me log every attempt to a lightweight datastore (even just a Sheet or a Dynamo table) before dispatching to any of the three, so a failure downstream doesn't lose the lead.

## Biggest Failure Point + Fallback

**HubSpot's default deduplication is on email, not phone** — and this form never collects email. So two patients (or the same patient submitting twice) with the same phone number but different names will create two separate contact records unless I explicitly override this. The fix: search HubSpot by a custom unique property (`phone_number`, normalized to E.164 format server-side) before writing, using the Contacts Search API, and `PATCH` the existing record if found instead of creating a new one. If two different people genuinely share a number (a real scenario with shared family phones in Indian healthcare lead-gen), I'd append the new enquiry as a **note/timeline entry** on the existing contact rather than overwrite the name field, so we don't silently lose either patient's identity.

The broader fallback: if the HubSpot call fails, queue the payload (e.g., SQS or a simple retry table) and retry with backoff, rather than dropping the lead — a failed CRM write shouldn't mean a patient never gets contacted.

## Monitoring the 2-Minute WhatsApp SLA

What could break it: Karix API rate limits or downtime, template message getting rejected by Meta's approval system, malformed phone numbers failing silently, or the serverless function itself cold-starting/timing out under a traffic spike from a new ad campaign. I'd monitor this with a simple synthetic check (a scheduled test submission every 15 minutes measuring actual delivery latency) plus real-time alerting (e.g., a Slack/email webhook) if the Karix API response time exceeds 30 seconds or returns a non-2xx status, so we know about an SLA breach before the patient does.
