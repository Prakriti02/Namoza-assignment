# Task 01 — GTM Event Schema: OrthoNow

## 1. Full Event Schema

| # | Event Name | Trigger Type (GTM) | Key Parameters (min. 3) | GA4 Report / Audience it Feeds |
|---|---|---|---|---|
| 1 | `clinic_page_view` | Page View trigger, condition: Page Path contains `/locations/` | `clinic_name`, `clinic_city`, `page_location` | Engagement → Pages and screens (per-clinic traffic); source audience "Location Interest – {city}" for geo remarketing |
| 2 | `call_now_click` | Click trigger (All Elements), condition: Click Classes contains `call-now-btn` | `click_text`, `page_location`, `clinic_name` (if on a location page) | Engagement → Events, marked as a Key Event; Google Ads phone-intent conversion |
| 3 | `whatsapp_click` | Click trigger, condition: Click URL contains `wa.me` OR Click ID equals `whatsapp-fab` | `click_url`, `page_location`, `device_category` | Engagement → Events; retargeting audience "WhatsApp Intent – No Booking" |
| 4 | `patient_guide_form_submit` | Form Submission trigger scoped to the gated-download form (or Custom Event if the form uses JS validation that blocks native submit) | `form_id`, `lead_source`, `page_location` | Engagement → Key events; feeds a lead-nurture email/WhatsApp audience |
| 5 | `patient_guide_download` | Click trigger on the actual PDF link — sequenced so it only fires after event #4 has fired in the same session (Trigger Group, or a dataLayer flag `guide_unlocked: true`) | `file_name`, `file_extension` (built-in), `page_location` | GA4's enhanced File Download event → Engagement report, content engagement |
| 6 | `booking_step_complete` (fires 3x with different `step_number`) | Custom Event trigger listening for dataLayer event name `booking_step_complete` | `step_number`, `step_name`, `clinic_location`, plus step-specific fields (see §2) | Explore → Funnel Exploration (booking funnel); Step 3 (`booking_confirmed`) is the one imported to Google Ads — see §3 |
| 7 | `blog_scroll_depth` | Native GTM **Scroll Depth** trigger, thresholds 25/50/75/90%, scoped to blog template pages | `percent_scrolled` (built-in), `page_location`, `article_category` | Engagement → Events; defines "Engaged Blog Reader" audience for content remarketing |

Notes on scoping:
- Events 1, 6, 7 are page/DOM-native and GTM can trigger them without any developer involvement beyond adding the GTM container and, for scroll depth, enabling the built-in variable.
- Events 2, 3, 4, 5 need CSS classes/IDs to be stable and unique in the DOM — flag this to the WordPress dev before go-live, since theme updates often strip custom classes.
- Event 6 is the one GTM **cannot** do on its own — see below.

---

## 2. Booking Form Funnel Tracking (3 steps)

**The core issue:** the booking form is a single-page, JS-driven multi-step form. GTM's triggers only see what's actually reflected in the DOM or the dataLayer — it has no native concept of "step 2 of a form." If the DOM doesn't fully re-render or the URL doesn't change between steps (which is standard for this kind of form), GTM has nothing to listen to unless the front-end explicitly tells it a step happened.

**So the dataLayer pushes must be written by the front-end developer**, at the exact moment each step's "Next" / "Confirm" button handler validates and advances the form. GTM's job is only to listen for those pushes via a Custom Event trigger — it does not, and cannot, detect the step change itself.

### GTM Trigger
One Custom Event trigger: **Event name equals `booking_step_complete`**. A single GA4 Event tag reads `step_number`, `step_name`, and the other dataLayer variables via Data Layer Variables, and fires for all three steps (the tag itself doesn't need to be duplicated — only the trigger condition is shared).

### DataLayer Push — Step 1 (Location + Specialty selected)
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

### DataLayer Push — Step 2 (Contact details entered)
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "{{clinic name}}",
  "preferred_date": "{{preferred date}}"
}
```

### DataLayer Push — Step 3 (Booking confirmed)
```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "booking_id": "{{booking id}}"
}
```

**Briefing the front-end dev, concretely:** "In the click handler for the Step 2 'Next' button, after your existing validation passes and before you transition the DOM to Step 3, add `window.dataLayer.push({event: 'booking_step_complete', step_number: 2, step_name: 'contact_details_entered', clinic_location: <value>, preferred_date: <value>})`. Don't fire it on button click alone — only after validation succeeds, otherwise we'll count failed attempts as completions."

### Surfacing drop-off in GA4 Funnel Exploration
1. Create a new **Funnel Exploration**.
2. Steps defined as: Step 1 = `booking_step_complete` where `step_number = 1`; Step 2 = same event where `step_number = 2`; Step 3 = same event where `step_number = 3`.
3. Turn on "Show elapsed time" and set the funnel to **open** (not strictly ordered) initially to sanity-check, then switch to **closed/sequential** for the real report so we only count genuine step-by-step progression.
4. GA4 will automatically compute drop-off % between each step — no separate "abandonment" event is needed, since the funnel is calculated from the presence/absence of the next step's event within the session. Segment this by `clinic_location` and `device_category` to see if drop-off is concentrated at a specific clinic or on mobile.

---

## 3. Conversion Action to Import into Google Ads

**Import: `booking_step_complete` where `step_number = 3` (i.e., `booking_confirmed`).**

Why this one over the others (e.g., `patient_guide_form_submit` or `call_now_click`):
- It's the closest proxy to actual business value (a completed appointment booking) rather than a soft, top-of-funnel signal.
- Importing a shallow event like a guide download or a call click would train Google Ads' bidding algorithms (Smart Bidding / tCPA) to optimize toward people who are merely curious, not people who book — this inflates lead volume while quietly raising real cost-per-booking.
- Call clicks are tempting because they feel high-intent, but they're noisy in this vertical (many are cold enquiries, wrong numbers, or duplicate calls from the same user) and hard to dedupe without call-tracking infrastructure, which isn't in place yet.
- Once call tracking and CRM-verified "Attended Appointment" data exist (via offline conversion import from HubSpot, phase 2), that becomes a stronger signal to graduate to — but for month one, `booking_confirmed` is the most reliable, already-available signal tied to revenue.
