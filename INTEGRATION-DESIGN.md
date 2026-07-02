# Task 03 — Integration Design
## OrthoNow Landing Page → HubSpot + WhatsApp + Google Ads

---

## End-to-End Architecture

When the patient submits the consultation form, the sequence is:

**Landing Page → Make (automation layer) → HubSpot API + Karix WhatsApp API + Google Ads Conversion**

### Why Make (formerly Integromat) over the alternatives

- **HubSpot native embed** is ruled out — we're not collecting email, which is HubSpot's default deduplication key. The native form embed has no hook to run deduplication logic on phone number before contact creation.
- **Zapier** can do this, but has no native Karix module and the workaround (HTTP module) costs an extra task per run. Make's HTTP module is included in the base plan.
- **Direct API call from the landing page** creates a CORS problem and exposes API keys in client-side JS. Never do this.
- **Make** gives us a single webhook receiver, sequential steps with conditional logic, error handling with retry, and a visual audit trail — all in one tool that the Namoza team can maintain without a developer for routine changes.

### Step-by-step flow

```
[Patient submits form]
       │
       ▼
[Landing page JS fires GTM dataLayer push]
       │
       ▼ (simultaneously, via fetch())
[Make Webhook — receives: name, phone, clinic_preference, page_url, timestamp]
       │
       ▼
[Make: Search HubSpot contacts by phone number]
       │
       ├── Phone EXISTS → Update existing contact (Name, Source, Lead Status)
       │
       └── Phone NOT FOUND → Create new contact:
                              Name, Phone, Clinic Preference,
                              Source = "Google Ads - Consultation Landing Page",
                              Lead Status = "New Enquiry"
       │
       ▼
[Make: HTTP POST to Karix WhatsApp API]
       Send template message to patient's phone within the Make flow
       (no separate scheduler needed — runs synchronously in the scenario)
       │
       ▼
[Make: HTTP GET to Google Ads Conversion Upload API]
       Upload offline conversion with gclid (passed from landing page URL param)
       OR rely on GA4 → Google Ads import if gclid is unavailable
```

---

## Phone Number Deduplication — The Critical Issue

HubSpot's default deduplication key is **email address**. Since our form collects only Name + Phone (no email), two patients submitting with the same phone number but different names will each create a **separate contact** under default HubSpot behaviour — duplicating the lead and potentially triggering two WhatsApp messages to the same number.

**Fix:** In Make, before creating a contact, run a **HubSpot Search Contacts** module filtered by `phone` property (using the `EQ` filter on the `phone` field). If a match is found, route to an **Update Contact** module instead of **Create Contact**. If no match, proceed to create. This is a manual deduplication step that replaces the email-based native logic.

**Additional safeguard:** Set `phone` as a unique property in HubSpot's property settings (Admin → Properties → Phone → check "unique value"). HubSpot will then reject duplicate phone numbers at the API level, giving Make a clear error signal to route to the update path instead.

---

## Biggest Failure Point & Fallback

The biggest failure point is the **Make webhook not receiving the form submission** — this can happen if the patient closes the browser immediately after clicking submit, if the `fetch()` call times out, or if Make has a service outage.

**Fallback:** 
1. The landing page also fires the GTM dataLayer push synchronously (client-side, before the fetch). This means Google Ads conversion tracking is never dependent on Make.
2. The `fetch()` to Make uses `keepalive: true` so the browser sends the request even if the page unloads immediately after submit.
3. In Make, enable **automatic retry** (3 attempts, 15-minute intervals) on the webhook scenario. Failed executions appear in the Make execution log with full payload — the Namoza team can manually re-trigger from there.
4. All form submissions are also stored client-side in `localStorage` as a last resort, so data is recoverable if needed.

---

## WhatsApp 2-Minute SLA

**What could break it:**
- Karix API rate limiting or downtime (most likely cause)
- Make scenario queue delay during traffic spikes (Make processes scenarios sequentially per connection; high volume can cause queuing)
- Patient's WhatsApp number not registered (message sent but never delivered — Karix returns a `delivered` vs `read` status)
- Template message not pre-approved by WhatsApp Business (messages outside approved templates are rejected instantly with no retry)

**How to monitor it:**
- In Make, add a timestamp variable at webhook receipt and another at Karix API response. Log the delta to a Google Sheet or a HubSpot custom property (`whatsapp_sent_at`). Alert if delta > 90 seconds.
- Set up a Make error handler on the Karix HTTP module: if status ≠ 200, send an email/Slack alert to the ops team with the patient's name and phone so a manual callback can happen within the SLA window.
- In Karix dashboard, monitor the delivery report webhook — undelivered messages (number not on WhatsApp) should trigger an SMS fallback via the same Karix account.

---

*Integration design v1.0 · OrthoNow onboarding · Namoza*
