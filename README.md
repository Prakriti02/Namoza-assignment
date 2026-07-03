# Namoza Developer Assignment — OrthoNow

Submission for: Developer – Position 1 (Client Web + Martech)

## Repo Contents

| Task | File | Description |
|---|---|---|
| 01 | [`task1-gtm-schema.md`](./task1-gtm-schema.md) | Full GTM event schema, booking-funnel dataLayer JSON, and the Google Ads conversion import decision |
| 02 | [`task2-landing-page/index.html`](./task2-landing-page/index.html) | Standalone HTML/CSS/JS landing page — no frameworks, no external requests |
| 03 | [`task3-integration-writeup.md`](./task3-integration-writeup.md) | HubSpot + WhatsApp (Karix) + Google Ads integration architecture |

## Before You Submit — Action Items for You

1. **PageSpeed screenshot (required, not optional):** I can't generate this for you — it has to be measured against a real, hosted URL.
   - Push this repo to GitHub, then enable **GitHub Pages** (Settings → Pages → deploy from `main` branch, `/task2-landing-page` folder or root).
   - Run the live URL through [PageSpeed Insights](https://pagespeed.web.dev/), Mobile tab.
   - Screenshot the score and drop it in `task2-landing-page/` as `pagespeed-mobile.png`.
   - The page is built with zero external requests (system fonts, inline SVGs, no images, no CDN scripts) specifically to make a 90+ score achievable — but only a real test on a real host counts.

2. **Verify the dataLayer push live**, since this is what they'll ask you to demo on the Loom:
   - Open `index.html` directly in a browser (double-click it — no server needed).
   - Open DevTools Console.
   - Fill the form with a valid name + 10-digit number starting 6–9, submit.
   - Type `dataLayer` in the console and confirm the `consultation_form_submitted` object is there with `form_id`, `lead_source`, and `page_location`.

3. **Loom (max 8 min)** — suggested structure:
   - 0:00–2:00 — Task 1: walk through the event table, then specifically explain *why* GTM can't natively detect form steps and how you'd brief the front-end dev (this is the question they're most likely to push on).
   - 2:00–5:00 — Task 2: live demo, show the console logging the push, show mobile view via DevTools device toolbar, show the PageSpeed score.
   - 5:00–8:00 — Task 3: talk through the architecture diagram in your head (or sketch it), and be ready for: *"If two patients submit with the same phone number but different names, what happens?"*

## A Note on How This Was Built

Parts of this submission were drafted with AI assistance (schema structure, boilerplate HTML/CSS/JS, writeup drafting). Before you present this:
- Make sure you can explain **every decision** in your own words — the brief is explicit that they'll ask follow-up questions on anything submitted.
- The two "hard filters" mentioned in the assignment brief are (a) understanding that multi-step form tracking requires a front-end dev to write dataLayer pushes — GTM does not do this automatically, and (b) knowing HubSpot dedupes on email by default, not phone. Both are addressed in the docs above, but you should be able to explain *why* they're true, not just recite the fact.
