# Task 01 — GTM Event Schema
## OrthoNow Full Event Tracking Implementation

---

## 1. Complete GTM Event Schema

| # | Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
|---|---|---|---|---|
| 1 | `booking_step_complete` | Custom Event (dataLayer push from front-end) | `step_number`, `step_name`, `clinic_location`, `specialty`, `session_id` | Funnel Exploration → Step-level drop-off; Audience: "Started but didn't complete booking" |
| 2 | `booking_confirmed` | Custom Event (dataLayer push on step 3 confirm) | `clinic_location`, `specialty`, `preferred_date`, `session_id` | Conversions report; Google Ads import target |
| 3 | `call_now_click` | Click — Just Links (href contains `tel:`) | `click_location` (homepage / clinic-page / landing-page), `clinic_name`, `page_path` | Events report; Audience: "High-intent callers" |
| 4 | `whatsapp_click` | Click — Just Links (href contains `wa.me`) | `click_location`, `page_path`, `device_category` | Events report; secondary conversion |
| 5 | `patient_guide_form_submit` | Custom Event (dataLayer push on gated form submit) | `form_name` = `patient_guide`, `lead_name`, `page_path` | Lead gen report; Audience: "Downloaded guide, not booked" |
| 6 | `patient_guide_download` | Click — Just Links (PDF file type) | `file_name`, `file_extension`, `page_path` | Events report; content engagement |
| 7 | `clinic_page_view` | Page View — URL contains `/clinic/` | `clinic_name` (from Page Path variable), `city`, `page_path` | Pages & Screens; Audience: "Viewed specific clinic" |
| 8 | `blog_scroll_depth` | Scroll Depth — 25%, 50%, 75%, 90% | `scroll_depth_threshold`, `article_title`, `page_path`, `read_time_seconds` | Engagement report; Content performance |
| 9 | `consultation_form_submitted` | Custom Event (dataLayer push on landing page form) | `lead_name`, `lead_phone_last4`, `page_path`, `traffic_source` | Conversions; Google Ads import (landing page specific) |

---

## 2. 3-Step Booking Form — Funnel Drop-off Tracking

### Architecture Note
GTM **cannot natively detect** interactions inside a multi-step form — there is no DOM event GTM can listen to that reliably fires when a user moves between in-page steps. Each `dataLayer.push()` below must be written by the **front-end developer** and triggered at the point in the application logic where the step is validated and the user advances. The GTM trigger is then a Custom Event listener that fires a GA4 event tag when each push is received.

Brief to front-end dev: *"At the moment the user successfully passes validation on step N and the UI advances to step N+1, call the dataLayer push shown below. Do not fire it on click — fire it after your validation function returns true."*

---

### Step 1 — Location & Specialty Selected

**GTM Trigger:** Custom Event · Event Name equals `booking_step_complete`  
**Condition:** `{{DLV - step_number}}` equals `1`

**dataLayer push (front-end writes this):**
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Koramangala",
  "specialty": "Knee & Sports Injuries",
  "session_id": "abc123xyz"
}
```

---

### Step 2 — Patient Details Entered

**GTM Trigger:** Custom Event · Event Name equals `booking_step_complete`  
**Condition:** `{{DLV - step_number}}` equals `2`

**dataLayer push (front-end writes this):**
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "Koramangala",
  "specialty": "Knee & Sports Injuries",
  "preferred_date": "2025-08-15",
  "session_id": "abc123xyz"
}
```

> **Note:** Do not push `name` or `phone` into the dataLayer. PII in GTM/GA4 violates Google's Terms of Service and creates a data compliance risk. Store those in your CRM only.

---

### Step 3 — Booking Confirmed

**GTM Trigger:** Custom Event · Event Name equals `booking_confirmed`

**dataLayer push (front-end writes this):**
```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmation",
  "clinic_location": "Koramangala",
  "specialty": "Knee & Sports Injuries",
  "preferred_date": "2025-08-15",
  "booking_id": "ORN-20250815-0042",
  "session_id": "abc123xyz"
}
```

---

### Surfacing Drop-off in GA4 Funnel Exploration

1. Go to **Explore → Funnel Exploration**
2. Set funnel type to **Open** (users can enter at any step)
3. Add steps:
   - Step 1: Event = `booking_step_complete` + Parameter `step_number` = `1`
   - Step 2: Event = `booking_step_complete` + Parameter `step_number` = `2`
   - Step 3: Event = `booking_confirmed`
4. Set **breakdown dimension** to `clinic_location` to see which clinic has the worst drop-off
5. Set **segment** to paid traffic only (Session source/medium contains `cpc`) to isolate campaign performance

The funnel will show % abandonment between each step. The highest drop-off step is where UX or copy needs fixing first.

---

## 3. Google Ads Conversion Import — Recommended Action

**Import: `booking_confirmed` (Step 3)**

**Reason:** This is the only event that represents completed intent — the patient has selected a clinic, entered details, and confirmed. Importing `booking_step_complete` (step 1 or 2) would tell Google's bidding algorithm to optimise toward users who start forms, not users who finish them. That inflates conversion volume but trains the algorithm on the wrong signal and will lower actual appointment rates.

`consultation_form_submitted` (landing page) is the second choice — useful for top-of-funnel optimisation on campaigns pointing to that specific page, but it captures intent one step earlier than a confirmed booking.

**Import method:** GA4 → Google Ads linked account → mark `booking_confirmed` as a conversion in GA4, then import via the Google Ads conversion import from GA4 (not a website tag — this avoids double-counting if GTM is also firing an Ads tag).

---

*Schema version 1.0 · Prepared for OrthoNow onboarding · Namoza*
