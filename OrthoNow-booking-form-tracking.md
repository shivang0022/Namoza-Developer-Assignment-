Booking Form Tracking (3-step) for OrthoNow

This file contains exact dataLayer JSON pushes for each booking step, GTM trigger mapping, and GA4 Funnel Exploration setup instructions.

GTM trigger & tag strategy (high level):
- Frontend pushes explicit dataLayer events at each step and on completion.
- GTM has Custom Event triggers that listen for the event value (e.g., booking_form_step, booking_form_complete).
- GA4 Event tags send those events and mapped parameters to the GA4 property.
- Use Within same session funnel scope in GA4 Funnel Exploration to measure step-to-step drop-off.
- Do NOT send raw PII. Use phone_present, name_present, or masked values.

Exact dataLayer pushes (JSON) — deliver these verbatim to frontend devs.

Step 1 — user selects clinic + specialty and lands on step 1 (push when step is shown or after selection):

{
  "event": "booking_form_step",
  "step": 1,
  "timestamp": "2026-06-30T12:00:00Z",
  "clinic_id": "clinic_009",
  "clinic_name": "OrthoNow Northside",
  "specialty": "orthopedics",
  "page_path": "/book",
  "user_pseudo_id": "GA_MEASUREMENT_PSEUDO_ID"
}

Step 2 — user enters name, phone, preferred date and navigates to step 2 (push on transition to step 2):

{
  "event": "booking_form_step",
  "step": 2,
  "timestamp": "2026-06-30T12:00:12Z",
  "clinic_id": "clinic_009",
  "clinic_name": "OrthoNow Northside",
  "specialty": "orthopedics",
  "form_name": "booking_contact",
  "pref_contact_method": "phone",
  "page_path": "/book",
  "user_pseudo_id": "GA_MEASUREMENT_PSEUDO_ID",
  "form_partial": {
    "name_present": true,
    "phone_present": true,
    "preferred_date_set": true
  }
}

Step 3 — confirmation preview (push when user reaches confirmation step):

{
  "event": "booking_form_step",
  "step": 3,
  "timestamp": "2026-06-30T12:00:28Z",
  "clinic_id": "clinic_009",
  "clinic_name": "OrthoNow Northside",
  "specialty": "orthopedics",
  "page_path": "/book/confirm",
  "user_pseudo_id": "GA_MEASUREMENT_PSEUDO_ID",
  "summary": {
    "name_masked": "J*** D***",
    "phone_masked": "+1-555-***-1234",
    "preferred_date": "2026-07-05"
  }
}

Booking complete — push after server confirmation or on client redirect after success:

{
  "event": "booking_form_complete",
  "timestamp": "2026-06-30T12:00:34Z",
  "booking_id": "BKG-20260630-001234",
  "clinic_id": "clinic_009",
  "clinic_name": "OrthoNow Northside",
  "specialty": "orthopedics",
  "user_pseudo_id": "GA_MEASUREMENT_PSEUDO_ID",
  "value": 0,
  "currency": "USD",
  "booking_status": "confirmed"
}

Error push example (use when validation fails or server returns an error):

{
  "event": "booking_form_error",
  "timestamp": "2026-06-30T12:00:10Z",
  "step": 2,
  "clinic_id": "clinic_009",
  "error_code": "VALIDATION_PHONE_MISSING",
  "error_message": "Phone number missing",
  "field_errors": ["phone"]
}

GTM variables & triggers (exact mapping):
- Create Data Layer Variables (dlv.step, dlv.clinic_id, dlv.specialty, dlv.booking_id, dlv.user_pseudo_id, dlv.value, dlv.booking_status).
- Triggers:
  - Booking Step 1 — Trigger Type: Custom Event -> Event name equals booking_form_step AND dlv.step equals 1.
  - Booking Step 2 — Custom Event where dlv.step equals 2.
  - Booking Step 3 — Custom Event where dlv.step equals 3.
  - Booking Complete — Custom Event -> Event name equals booking_form_complete.
  - Booking Error — Custom Event -> Event name equals booking_form_error.
- Tags:
  - GA4 Event tag for booking_form_step: Event Name booking_form_step, send parameters step, clinic_id, specialty, page_path, form_partial.name_present, form_partial.phone_present.
  - GA4 Event tag for booking_form_complete: Event Name booking_form_complete, send booking_id, clinic_id, specialty, value, currency, booking_status.
  - GA4 Event tag for booking_form_error: Event Name booking_form_error, send step, error_code, error_message.

GA4 Funnel Exploration setup (to measure step-level drop-off):
1. In GA4, go to Explore -> Funnel exploration -> Create new.
2. Set Funnel type: Trended or Standard (Standard recommended for visualizing step drops in-session).
3. Use events (ordered) as steps:
   - Step 1: Include booking_form_step where parameter step equals 1.
   - Step 2: Include booking_form_step where parameter step equals 2.
   - Step 3: Include booking_form_step where parameter step equals 3.
   - (Optional) Final Step: booking_form_complete to show confirmed bookings.
4. Scope: Within the same session (default).
5. Add breakdowns: clinic_id, specialty, page_path.
6. Add segment comparisons: session_medium or utm_campaign to compare paid vs organic traffic.

How to surface step-level drop-off in GA4 reports:
- Use the Funnel Exploration above for visual step drop-offs.
- Build an Audience booking_funnel_dropouts_step1_to_2 where users had step==1 but not step==2 within session — use for remarketing.
- Create custom explorations with clinic_id and traffic_source to prioritize fixes for high-volume clinics with high drop-off.

Google Ads conversion to import (recommendation):
- Import booking_form_complete as the primary conversion to Google Ads.
- Rationale: It is the most reliable indicator of business value (confirmed appointment). It reduces noise from partial form completions, clicks, or downloads and aligns paid spend with actual bookings.
- Implementation notes: Mark booking_form_complete as conversion in GA4, ensure the event has stable booking_id or server-side confirmation, then link GA4 to Google Ads and import the conversion.

If you'd like, I can also export a GTM Implementation Checklist, produce a README for dev handoff with dataLayer.push() snippets in JS, or draft GTM Trigger definitions.