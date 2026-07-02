# GTM Event Schema & Tracking Architecture Specification
**Client:** OrthoNow (9 Orthopaedic Clinics in Bengaluru, Hyderabad, Chennai)  
**Target Audience:** Working professionals (Ages 28–50) in Bengaluru experiencing knee or back pain  
**Document version:** 1.0.0  

---

## SECTION A — Complete GTM Event Schema Table

| Event Name | Trigger Type | Key Parameters (min 3) | GA4 Report / Audience |
| :--- | :--- | :--- | :--- |
| `booking_step_complete` *(Step 1)* | Custom Event Trigger | 1. `step_number` (Integer: `1`) <br>2. `step_name` (String: `'location_specialty_selected'`) <br>3. `clinic_location` (String: `'Koramangala, Bengaluru'`) <br>4. `specialty` (String: `'Knee Replacement'`) <br>5. `session_id` (String: `'172000213.98765'`) | **Funnel Exploration** -> Measure the flow of users from selecting location/specialty to entering contact details, pinpointing where friction is highest. |
| `booking_step_complete` *(Step 2)* | Custom Event Trigger | 1. `step_number` (Integer: `2`) <br>2. `step_name` (String: `'contact_info_entered'`) <br>3. `preferred_date` (String: `'2026-07-05'`) <br>4. `session_id` (String: `'172000213.98765'`) | **Funnel Exploration** -> Analyze friction at the contact entry stage. Helps determine if requesting personal details drops engagement. |
| `booking_step_complete` *(Step 3)* | Custom Event Trigger | 1. `step_number` (Integer: `3`) <br>2. `step_name` (String: `'booking_confirmed'`) <br>3. `booking_id` (String: `'ON-2026-88741'`) <br>4. `clinic_location` (String: `'Koramangala, Bengaluru'`) <br>5. `specialty` (String: `'Knee Replacement'`) <br>6. `session_id` (String: `'172000213.98765'`) | **Audience Builder** -> Build a "Conversions - Booked Patients" audience to exclude them from remarketing campaigns, saving ad spend. |
| `booking_form_abandonment` | Custom Event Trigger | 1. `abandoned_step` (Integer: `2`) <br>2. `clinic_location` (String: `'Indiranagar, Bengaluru'`) <br>3. `specialty` (String: `'Lumbar Spine Herniation'`) <br>4. `time_spent_seconds` (Integer: `52`) | **Audience Builder** -> Build a remarketing list of high-intent "Form Abandoners" to retarget with specific WhatsApp-first quick-booking ads. |
| `call_now_click` | Click - Just Links Trigger | 1. `click_location` (String: `'sticky_header'`) <br>2. `phone_number` (String: `'+919845012345'`) <br>3. `page_path` (String: `'/bengaluru/knee-pain-consultation'`) | **Event Report** -> Identify if mobile-first working professionals prefer direct calls during peak office hours rather than filling forms. |
| `whatsapp_click` | Click - All Elements Trigger | 1. `click_location` (String: `'floating_widget'`) <br>2. `target_url` (String: `'https://wa.me/919845012345'`) <br>3. `page_path` (String: `'/bengaluru/back-pain-specialist'`) | **Path Exploration** -> Determine if users choose WhatsApp after reviewing doctor profiles or FAQs, validating high-intent micro-conversions. |
| `patient_guide_form_submit` | Custom Event Trigger | 1. `form_name` (String: `'patient_guide_download'`) <br>2. `guide_title` (String: `'Bengaluru Working Professional Back Pain Relief Manual'`) <br>3. `page_path` (String: `'/bengaluru/back-pain-specialist'`) | **Event Report** -> Track top-of-funnel (TOFU) content download conversions to evaluate email/SMS follow-up nurture success. |
| `patient_guide_download_complete` | Click - Just Links Trigger | 1. `file_name` (String: `'orthonow-bengaluru-back-pain-relief-guide.pdf'`) <br>2. `file_extension` (String: `'pdf'`) <br>3. `click_text` (String: `'Download Free Guide Now'`) | **Event Report** -> Audit file downloads to ensure users successfully retrieve the PDF assets after completing the form. |
| `clinic_page_view` | Page View Trigger | 1. `clinic_location` (String: `'Whitefield, Bengaluru'`) <br>2. `city` (String: `'Bengaluru'`) <br>3. `page_path` (String: `'/clinics/whitefield'`) | **Event Report** -> Assess geographic distribution of organic interest across OrthoNow's 9 clinics to align operational capacities. |
| `blog_scroll_depth` | Scroll Depth Trigger | 1. `scroll_depth_threshold` (Integer: `75`) <br>2. `article_title` (String: `'Sitting Posture and Sciatica: Guidelines for Bengaluru IT Workers'`) <br>3. `page_path` (String: `'/blog/sciatica-guidelines-it-workers'`) | **Path Exploration / Event Report** -> Measure depth thresholds (25%, 50%, 75%, 90%) using a single event and custom parameter. This avoids event bloat, respects GA4 limits, and makes it simple to analyze drop-offs in blog article engagement. |

---

## SECTION B — Booking Form Step-Level Funnel Tracking

### 1. Trigger Strategy by Step
* **Step 1 (Select clinic location + specialty):** 
  * **Trigger Type:** Custom Event Trigger (`booking_step_complete` where `step_number` equals `1`).
  * **Justification:** Triggering on a click of the "Next" button is unreliable because it fires even if validation fails (e.g., if a user doesn't pick a location). A Custom Event ensures that only validated selections trigger the event.
* **Step 2 (Enter name / phone / preferred date):**
  * **Trigger Type:** Custom Event Trigger (`booking_step_complete` where `step_number` equals `2`).
  * **Justification:** Similar to Step 1, contact details must pass client-side regex validations (e.g., verifying a valid 10-digit Indian mobile number format). Using a Custom Event prevents false tracking of failed submission attempts.
* **Step 3 (Confirm booking — final submission):**
  * **Trigger Type:** Custom Event Trigger (`booking_step_complete` where `step_number` equals `3`).
  * **Justification:** A simple button click on "Confirm Booking" doesn't guarantee the CRM API accepted the lead. If the server fails, a button-click conversion would lead to inflated counts in GA4. The trigger must listen for a custom success event pushed *inside* the API's success callback.

### 2. Implementation Briefing Document for Front-End Developers
* **Responsibility:** The front-end developer writes the JavaScript to push data to the `window.dataLayer` object. The GTM developer designs GTM variables, triggers, and tags to capture this pushed data.
* **Where in Code to Fire:**
  * **Step 1 & 2:** Immediately after the client-side validation logic passes, right before the DOM transitions to the next step panel.
  * **Step 3:** Inside the asynchronous `fetch()` or AJAX success callback function (HTTP status `200 OK`) that processes the CRM API booking request.
* **What NOT to Do:**
  * **Do NOT** fire pushes directly on `onclick` HTML event handlers of the "Next" buttons before form validation is complete.
  * **Do NOT** re-initialize `window.dataLayer = [];` inside the code as it destroys the global GTM tracking instance. Use `window.dataLayer = window.dataLayer || [];` instead.
  * **Do NOT** push Personally Identifiable Information (PII) like names, raw phone numbers, or email addresses directly to GA4 parameters. It violates Google’s terms of service and can lead to account suspension. Keep parameters metadata-focused.

### 3. Step-Level dataLayer Push JSON Specs

#### Step 1 JSON
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement",
  "timestamp": "2024-01-15T10:30:00+05:30",
  "session_id": "172000213.98765"
}
```

#### Step 2 JSON
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_info_entered",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement",
  "preferred_date": "2026-07-02",
  "timestamp": "2026-07-01T12:45:00+05:30",
  "session_id": "172000213.98765"
}
```

#### Step 3 JSON
```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "booking_id": "ON-2026-98745",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement",
  "preferred_date": "2026-07-02",
  "timestamp": "2026-07-01T12:46:00+05:30",
  "session_id": "172000213.98765"
}
```

### 4. GA4 Funnel Exploration Configuration Steps
1. **Navigate:** Open GA4, select **Explore** on the left menu, and click **Funnel Exploration**.
2. **Step 1 Condition:** Name the step "Step 1: Location & Specialty". Select Event: `booking_step_complete` -> Add Parameter `step_number` equals `1`.
3. **Step 2 Condition:** Name the step "Step 2: Contact Info". Select Event: `booking_step_complete` -> Add Parameter `step_number` equals `2`.
4. **Step 3 Condition:** Name the step "Step 3: Confirmed Booking". Select Event: `booking_step_complete` -> Add Parameter `step_number` equals `3`.
5. **Breakdown:** Drag the Custom Dimension `clinic_location` or `specialty` to the **Breakdown** section.
6. **Insight Generated:** This visualization reveals step-level drop-offs across orthopaedic clinics in Bengaluru (e.g., higher dropout at Step 2 in Whitefield compared to Koramangala). This tells performance marketing teams where to run clinic-specific localized search ads, refine copy, or check localized scheduling options.

---

## SECTION C — Google Ads Conversion Action

We will import `booking_step_complete` with parameter `step_number` = `3` (Final Booking Submission) as our primary conversion action. 

Step 1 and Step 2 represent intermediate intent and are not monetization events. Call Now or WhatsApp widget clicks are micro-conversions; they are prone to click-spam, cannot easily deduplicate multiple clicks from the same user, and don't confirm if a lead was successfully captured in the database. 

We will use a **"One"** (one conversion per click) match type, as multiple submissions by the same user for the same injury (e.g., knee pain) do not represent unique conversion values. 

The conversion window is set to **30 days**. Orthopaedic consultations for debilitating knee or spinal pain are high-involvement healthcare decisions with an average research period of 7–14 days. 

This conversion action enables a **Target CPA (Cost Per Acquisition)** smart bidding strategy, allowing Google’s algorithms to optimize bid adjustments in real-time toward users with high-intent signals likely to finish the booking process.

---

## SECTION D — Front-End Developer Brief

### Understanding the dataLayer
The `dataLayer` is a structured JavaScript array (`window.dataLayer`) used by Google Tag Manager to consume page data, user events, and variables in a robust way, decoupled from the DOM. Without it, GTM would have to rely on fragile DOM scraping (like reading button classes), which breaks whenever the HTML layout changes.

### Trigger Timing Requirements
You must push events only at specific, validated states:
1. **Step Transitions:** Push immediately *after* all input fields on the current step pass client-side regex checks (such as verifying a 10-digit Indian mobile number format) and before displaying the next screen.
2. **Final Submission:** Push *inside* the successful response handler of your server-side API call (e.g., inside the `.then(res => res.json())` block where `res.ok` is true). Do not push on the button click, as network errors would result in phantom conversions in analytics.

### Vanilla JS Multi-Step Form Implementation Example
```javascript
// Initialize the dataLayer if it doesn't already exist
window.dataLayer = window.dataLayer || [];

// 1. Step 1: Location & Specialty validation and transition
function handleStep1Submit() {
  const clinic = document.getElementById('clinic_select').value;
  const specialty = document.getElementById('specialty_select').value;
  
  if (!clinic || !specialty) {
    showError("Please select both a clinic location and specialty.");
    return;
  }
  
  // Validation succeeded, push to dataLayer
  window.dataLayer.push({
    "event": "booking_step_complete",
    "step_number": 1,
    "step_name": "location_specialty_selected",
    "clinic_location": clinic,
    "specialty": specialty,
    "timestamp": new Date().toISOString(),
    "session_id": getClientSessionId() // Helper returning GA client ID
  });
  
  // Transition UI to Step 2
  navigateToStep(2);
}

// 2. Step 2: Contact Info validation and transition
function handleStep2Submit() {
  const name = document.getElementById('patient_name').value.trim();
  const phone = document.getElementById('patient_phone').value.trim();
  const date = document.getElementById('preferred_date').value;
  
  const phoneRegex = /^(?:\+91|0)?[6-9]\d{9}$/;
  if (!name || !phoneRegex.test(phone) || !date) {
    showError("Please enter a valid name, 10-digit phone number, and future date.");
    return;
  }
  
  // Validation succeeded, push to dataLayer
  window.dataLayer.push({
    "event": "booking_step_complete",
    "step_number": 2,
    "step_name": "contact_info_entered",
    "preferred_date": date,
    "timestamp": new Date().toISOString(),
    "session_id": getClientSessionId()
  });
  
  // Transition UI to Step 3 (Confirmation panel)
  navigateToStep(3);
}

// 3. Step 3: Final CRM Submission
function handleFinalBooking() {
  const payload = {
    clinic: document.getElementById('clinic_select').value,
    specialty: document.getElementById('specialty_select').value,
    name: document.getElementById('patient_name').value.trim(),
    phone: document.getElementById('patient_phone').value.trim(),
    date: document.getElementById('preferred_date').value
  };
  
  // Disable button to prevent double-submissions
  setLoadingState(true);
  
  fetch('https://api.orthonow.in/v1/bookings', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload)
  })
  .then(response => {
    if (!response.ok) throw new Error('API submission failed');
    return response.json();
  })
  .then(data => {
    // API SUCCESS - Push final conversion event
    window.dataLayer.push({
      "event": "booking_step_complete",
      "step_number": 3,
      "step_name": "booking_confirmed",
      "booking_id": data.booking_id, // Returned by database
      "clinic_location": payload.clinic,
      "specialty": payload.specialty,
      "preferred_date": payload.date,
      "timestamp": new Date().toISOString(),
      "session_id": getClientSessionId()
    });
    
    // Switch to Thank You screen
    showThankYouScreen(data.booking_id);
  })
  .catch(error => {
    console.error("Booking error:", error);
    showError("Could not confirm your booking. Please try again or Call Now.");
  })
  .finally(() => {
    setLoadingState(false);
  });
}
```

### Common Developer Mistakes to Avoid
* **Mistake 1: Initializing as an Object:** Writing `window.dataLayer = {};` instead of `window.dataLayer = window.dataLayer || [];`. GTM expects an Array; setting it to an Object breaks all built-in listeners.
* **Mistake 2: Missing Validation Check:** Firing the step events directly on `click` triggers instead of after validation has succeeded. This leads to massive inflation of event metrics.
* **Mistake 3: Hardcoding Variable Values:** Using static values like `'Koramangala'` instead of passing dynamic inputs picked by the patient in the select options or input fields.
* **Mistake 4: Not Handling Retry Logic/Errors:** Firing the step 3 event before the API successfully responds. If the server fails, the event is already in the dataLayer and GTM cannot pull it back.
* **Mistake 5: Pushing PII:** Including raw phone numbers or names in the dataLayer push parameters. GA4 will flag this and block accounts. Keep PII in your API request payload, never in the analytics dataLayer.
