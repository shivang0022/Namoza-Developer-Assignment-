OrthoNow GTM Event Schema

This document lists the full GTM event schema to implement across OrthoNow before paid campaigns go live.

Event Name | Trigger Type (GTM) | Key Parameters (min 3) | GA4 Report / Audience
---|---|---|---
booking_form_step | Custom Event (dataLayer push) | step (1|2|3), clinic_id, specialty, page_path, user_pseudo_id | Funnel Exploration (ordered steps). Audience: engaged_bookers (users reaching step ≥2)
booking_form_submit | Form Submit / Click | step, form_success (true/false), validation_errors (count), clinic_id | Events / Engagement. Audience: form_submitters
booking_form_complete | Custom Event (dataLayer push on confirmation) | booking_id, clinic_id, specialty, value, currency, booking_status | Conversions / Purchase-like. Audience: converters_booking_confirmed (import to Google Ads)
booking_form_error | Custom Event (on validation/error) | step, error_code, error_message, field_errors | Debugging events; Audience: booker_errors
call_now_click | Click (GTM Click - tel: links/buttons) | clinic_id, phone_number, page_location (homepage/clinic/landing), cta_position | Events / Engagement; Audience: callers
whatsapp_click | Click (widget or outbound wa.me) | wa_number, page_path, widget (floating/gated), utm_campaign | Events; Audience: whatsapp_inquiries
pdf_download_view | Click / Gated Form View | file_name, file_path, form_gated (true), page_path | Content Events; Audience: downloaders_funnel_start
pdf_download_submit | Form Submit (gated download) | file_name, user_name_present (true/false), phone_present (true/false), consent (gdpr) | Events / Conversions; Audience: guide_downloaders
pdf_download | Link Click (PDF GET) | file_name, download_method (direct/gated), clinic_interest | Content Events; Audience: downloaders_final
clinic_location_view | Page View (path match /clinic/*) | clinic_id, clinic_name, page_path, referrer | Pages & Screens / Locations; Audience: clinic_page_visitors
blog_article_view | Page View (article page) | article_id, author, category, page_path | Pages & Screens / Content; Audience: readers
scroll_depth | Scroll trigger (25/50/75/100%) | percent_scrolled, article_id, page_path, engaged_time | Engagement Events; Audience: engaged_readers (>=75% or >=60s)
cta_click | Click (site CTA selectors) | cta_id, cta_text, page_path, position | Events; Audience: cta_clickers
outbound_link_click | Click (external links) | outbound_url, domain, page_path, link_text | Events; Audience: outbound_leads
phone_number_copy | Click (copy-to-clipboard) | phone_number, clinic_id, page_path | Engagement events measuring intent
page_engaged | Timer / Heartbeat | engaged_seconds, page_path, user_pseudo_id | Engagement metrics; Audience: highly_engaged

Implementation notes (summary):
- All custom events must be pushed to dataLayer as objects, then GTM listens with Custom Event triggers.
- Map parameters to GA4 Event tag fields in GTM; use GA4 recommended event names where sensible but preserve custom names for clarity.
- Avoid sending raw PII to GA4. Use presence flags (e.g., phone_present) or hashed identifiers.
- Use consistent clinic_id/article_id naming and types across the site for clean reporting and audience building.
- Mark booking_form_complete (or server-side equivalent) as a conversion in GA4 and import to Google Ads.

Next: deliver booking form tracking text file with exact dataLayer pushes.