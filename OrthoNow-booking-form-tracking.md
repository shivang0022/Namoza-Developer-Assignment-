# OrthoNow Booking Form Tracking (3-step)

This document defines the booking form tracking implementation for OrthoNow.
It includes exact `dataLayer` JSON pushes for each booking step, GTM trigger/tag mapping, and GA4 Funnel Exploration setup instructions.

> Important: Do not send raw PII. Use flags like `phone_present`, `name_present`, or masked values only.

## 1. Strategy Overview

- Frontend pushes explicit `dataLayer` events at each step and on completion.
- GTM listens for those events using Custom Event triggers.
- GA4 Event tags send the mapped event data to the GA4 property.
- Use a "Within same session" funnel scope in GA4 Funnel Exploration to measure step-to-step drop-off.

## 2. Exact `dataLayer` Pushes

### 2.1 Step 1

User selects clinic + specialty and lands on step 1.
Push this event when step 1 is shown or after the selection is complete.

```json
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
```

### 2.2 Step 2

User enters name, phone, and preferred date and moves to step 2.
Push this event on transition to step 2.

```json
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
```

### 2.3 Step 3

User reaches the confirmation preview step.
Push this event when the confirmation step is displayed.

```json
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
```

### 2.4 Booking Complete

Push after server confirmation or on client redirect after success.

```json
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
```

### 2.5 Error Event

Use this push when validation fails or the server returns an error.

```json
{
  "event": "booking_form_error",
  "timestamp": "2026-06-30T12:00:10Z",
  "step": 2,
  "clinic_id": "clinic_009",
  "error_code": "VALIDATION_PHONE_MISSING",
  "error_message": "Phone number missing",
  "field_errors": ["phone"]
}
```

## 3. GTM Variables, Triggers, and Tags

### 3.1 Data Layer Variables

Create the following GTM Data Layer Variables:

- `dlv.step`
- `dlv.clinic_id`
- `dlv.specialty`
- `dlv.booking_id`
- `dlv.user_pseudo_id`
- `dlv.value`
- `dlv.booking_status`

### 3.2 Triggers

- **Booking Step 1**
  - Trigger Type: Custom Event
  - Event name equals `booking_form_step`
  - Condition: `dlv.step` equals `1`

- **Booking Step 2**
  - Trigger Type: Custom Event
  - Event name equals `booking_form_step`
  - Condition: `dlv.step` equals `2`

- **Booking Step 3**
  - Trigger Type: Custom Event
  - Event name equals `booking_form_step`
  - Condition: `dlv.step` equals `3`

- **Booking Complete**
  - Trigger Type: Custom Event
  - Event name equals `booking_form_complete`

- **Booking Error**
  - Trigger Type: Custom Event
  - Event name equals `booking_form_error`

### 3.3 Tags

- **GA4 Event tag** for `booking_form_step`
  - Event Name: `booking_form_step`
  - Parameters: `step`, `clinic_id`, `specialty`, `page_path`, `form_partial.name_present`, `form_partial.phone_present`

- **GA4 Event tag** for `booking_form_complete`
  - Event Name: `booking_form_complete`
  - Parameters: `booking_id`, `clinic_id`, `specialty`, `value`, `currency`, `booking_status`

- **GA4 Event tag** for `booking_form_error`
  - Event Name: `booking_form_error`
  - Parameters: `step`, `error_code`, `error_message`

## 4. GA4 Funnel Exploration Setup

Use GA4 Explore > Funnel exploration to measure step-level drop-off.

1. Create a new Funnel exploration.
2. Set Funnel type to Standard (recommended) or Trended.
3. Configure ordered steps:
   - Step 1: `booking_form_step` where parameter `step` equals `1`
   - Step 2: `booking_form_step` where parameter `step` equals `2`
   - Step 3: `booking_form_step` where parameter `step` equals `3`
   - Optional final step: `booking_form_complete`
4. Set scope to Within the same session.
5. Add breakdowns for `clinic_id`, `specialty`, and `page_path`.
6. Add segment comparisons for `session_medium` or `utm_campaign`.

## 5. Reporting and Audiences

- Use the funnel exploration for visual step drop-offs.
- Build an audience such as `booking_funnel_dropouts_step1_to_2` for users who had `step == 1` but not `step == 2` within the same session.
- Create custom explorations by `clinic_id` and traffic source to prioritize high-volume clinics with high drop-off.

## 6. Google Ads Conversion Recommendation

- Import `booking_form_complete` as the primary conversion in Google Ads.
- This event is the most reliable indicator of confirmed appointments.
- Ensure the event has a stable `booking_id` or server-side confirmation before importing.
- Link GA4 to Google Ads and import the conversion.

---

If you want, I can also create a GTM implementation checklist, a developer handoff README with `dataLayer.push()` examples, or draft the GTM trigger definitions.