# Integration & CRM Architecture Design
**Client:** OrthoNow Clinics  
**Author:** Namoza Growth Team  
**Document Version:** 1.0.0  

---

## PART A — Integration Architecture

To connect the landing page form, HubSpot, and Karix WhatsApp, we deploy an API Middleware (Node.js serverless function on AWS Lambda). We choose this custom middleware over HubSpot’s native form embed or Zapier. Native embeds lack custom routing logic, while Zapier introduces latency and high costs for conditional lookups.

### Numbered Sequence:
1. Patient submits landing page form. Client-side validation succeeds.
2. Page calls API Middleware via a secure `POST` request.
3. Middleware queries HubSpot Contacts Search API using the phone number.
4. If found, Middleware updates the existing contact ID; if not, it creates a new contact.
5. HubSpot confirms contact creation/update via HTTP 200/201.
6. Middleware parses the response and immediately calls the Karix WhatsApp API with template variables.
7. Karix delivers the confirmation template to the patient.

The primary failure point is HubSpot/Karix API rate limits or network timeouts. To handle this, we implement a fallback pipeline using AWS SQS (Simple Queue Service) and AWS DynamoDB. If a webhook or API call fails, the event is saved in a DynamoDB "failed_leads" table. An SQS dead-letter queue (DLQ) triggers retry attempts (exponential backoff up to 5 times). If it fails permanently, a Lambda function pushes a critical alert to the operations team's Slack channel via a webhook and sends an SMS alert via Twilio, ensuring zero lead loss.

SLA breaches occur due to:
* **Karix API outages:** Monitored by pinging Karix status endpoints every 30 seconds; alerts fire if response exceeds 1500ms.
* **Queue bottlenecks:** Monitored via AWS CloudWatch alerts when SQS age-of-oldest-message exceeds 60 seconds.
* **Invalid numbers:** Detected by parsing Karix callback payloads (e.g., error code 101 for invalid MSISDN) and writing them directly to HubSpot's `hs_lead_status` as "Invalid Number".

Because HubSpot default deduplication relies on email, omitting email causes duplicated contacts. We resolve this by performing a `POST /crm/v3/objects/contacts/search` lookup filtering by `phone`. If a match occurs, we use `PATCH /crm/v3/objects/contacts/{contactId}` to update the contact, rather than creating a new record. This custom search-before-create logic confirms our choice of a Node.js API Middleware over native forms, which cannot execute search queries.

---

## PART B — Architecture Diagram (ASCII)

```
Landing Page Form Submit
        │
        ▼ [HTTPS POST Payload]
API Middleware (Node.js on AWS Lambda)
        │
        ▼ [POST /crm/v3/objects/contacts/search]
HubSpot Contacts Search (Phone Lookup)
        │
        ├─► [Match Found] ──► PATCH /crm/v3/objects/contacts/{id} (Update Contact)
        │
        └─► [No Match] ─────► POST /crm/v3/objects/contacts (Create Contact)
        │
        ▼ [API Success Callback]
API Middleware (Node.js on AWS Lambda)
        │
        ▼ [POST /v1/message/send/template]
Karix WhatsApp API
        │
        ▼ [Direct Delivery via WhatsApp Business API]
Patient receives message ✓
```

---

## PART C — HubSpot Contact Properties Table

| Property Name | Property Type | Value | Source |
| :--- | :--- | :--- | :--- |
| `firstname` | Single Line Text | User's Name (e.g., `'Pranav Patel'`) | Form input field (`#patient_name`). |
| `phone` | Single Line Text | User's Mobile (e.g., `'+919845098765'`) | Form input field (`#patient_phone`). *Note: Crucial for the Search API middleware lookup since email is not collected.* |
| `ortho_clinic_preference` | Dropdown Select | Dynamic Clinic Location (e.g., `'Koramangala, Bengaluru'`) | Extracted dynamically from page configuration context or defaults. |
| `lead_source` | Single Line Text | `'google_ads_consultation_lp'` | Automatically hardcoded inside form submit handler payload. |
| `hs_lead_status` | Dropdown Select | `'NEW'` | Automated default state pushed by middleware on lead creation. |
| `last_form_submitted` | Single Line Text | `'ortho_consultation_lp'` | Set automatically to tag the source asset. |
| `ortho_pain_focus` | Dropdown Select | Custom property mapping pain area (e.g., `'Knee'`) | Parsed by middleware based on URL query parameters or page variant. |

---

## PART D — WhatsApp Message Template

* **Approved Template Name:** `orthonow_consultation_confirmation`
* **Status:** Pre-Approved (Karix/WhatsApp Business Registry)
* **Category:** Utility
* **Language:** English (en)

### Template Body Content:
```text
Hello {{1}},

Thank you for choosing OrthoNow. We have received your request for a priority orthopaedic consultation.

Here is what happens next:
1. Our Senior Medical Advisor will call you within 15 minutes at this number to discuss your joint or back pain symptoms.
2. We will match you with the right specialist (knee, spine, or sports medicine) and confirm your appointment slot at your preferred Bengaluru clinic.

If you want to speed up your booking or share diagnostic reports (X-Ray/MRI) ahead of time, click below to chat directly with our care coordinator.

We are committed to helping you walk pain-free.

Best regards,
Patient Care Team
OrthoNow Clinics
```

### Template Interactive Button:
* **Button Type:** Quick Reply / Call to Action (Dynamic URL)
* **Button Label:** Chat with Care Coordinator
* **Target URL:** `https://wa.me/918049123456?text=Hi%20OrthoNow%2C%20please%20connect%20me%20with%20a%20clinical%20care%20coordinator.`
