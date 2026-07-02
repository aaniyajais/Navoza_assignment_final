# Task 01 — GTM Event Schema — OrthoNow

## Assignment Approach

Before designing the event schema, I first identified OrthoNow’s core business goal: converting website visitors into confirmed clinic bookings.

Based on that, I prioritized tracking high-intent user actions such as:
- clinic exploration
- call / WhatsApp engagement
- booking progression
- confirmed consultations

The schema below is designed to answer three business questions:
1. Which traffic sources generate high-quality leads?
2. Where do users drop off in the booking funnel?
3. Which clinics and specialties attract the strongest demand?

## 1. Full Event Schema

| Event Name | Page Type | GTM Trigger Type | Key Parameters | GA4 Report |
|---|---|---|---|---|
| `page_view`¹ | All Pages | Page View | `page_name`, `page_url`, `traffic_source` | Traffic Acquisition |
| `clinic_page_view` | Clinic Pages | Page View / History Change (URL matches `/clinics/*`) | `clinic_id`, `clinic_city`, `page_location` | Pages & Screens, Geo Remarketing |
| `call_now_click` | All Pages | Click – Just Links (`href` contains `tel:`) | `page_location`, `clinic_location`, `button_position` | Engagement > Events, Audience: High Intent Leads |
| `whatsapp_chat_open` | All Pages | Click – Just Links (`href` contains `wa.me`) | `page_location`, `entry_point`, `device_category` | Engagement > Events, Audience: WhatsApp Intent Leads |
| `booking_started` | Main Website Booking Form | Custom Event (form opened / first field interaction) | `page_location`, `clinic_location`, `device_category` | Funnel Exploration (entry point) |
| `booking_step_complete` *(step_number: 1)* | Main Website Booking Form | Custom Event (dataLayer push on step change) | `step_number`, `clinic_location`, `specialty`² | Funnel Exploration |
| `booking_step_complete` *(step_number: 2)* | Main Website Booking Form | Custom Event (dataLayer push on step change) | `step_number`, `lead_id`, `device_type` | Funnel Exploration |
| `booking_confirmed` | Main Website Booking Form | Custom Event (dataLayer push on final confirmation) | `lead_id`, `clinic_location`, `booking_status` | Conversions report, **Google Ads import**, Audience: Confirmed Bookings |
| `patient_guide_form_submitted` | Guide Form | Custom Event (gated form success) | `form_name`, `guide_topic`, `page_location` | Lead Magnet Funnel |
| `patient_guide_download` | Guide Form | Custom Event (PDF successfully downloaded) | `guide_topic`, `page_location`, `device_category` | Engagement > Events, Audience: Content Downloaders |
| `blog_scroll_depth` | Blog | Scroll Depth (25/50/75/90%) | `scroll_percentage`, `page_location`, `article_category` | Engagement Report, Audience: Engaged Blog Readers |

¹ *GA4's Enhanced Measurement already fires `page_view` automatically. This row represents the same event enriched with custom parameters — not a duplicate fire.*

² *`specialty` is kept for funnel completeness but treated as operational data — never combined with name or phone number inside a single GA4 hit. That pairing stays inside HubSpot only, to avoid sending anything PII-adjacent into GA4.*

---

## 2. Booking Form — Step-by-Step Funnel Tracking

**The core issue:** GTM cannot automatically understand internal state changes inside a multi-step booking form. It only reacts to page loads, clicks, and DOM events. Since steps 1 → 2 → 3 happen dynamically without a page reload or URL change, the frontend must explicitly push dataLayer events whenever a step is completed. My role here is to define the event contract below; the frontend developer’s role is to fire these pushes at the correct moments inside the booking flow.
### Step 1 — Location + Specialty selected
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Sports Injury"
}
```

### Step 2 — Contact details entered
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "lead_id": "ORTHO-8842",
  "device_type": "mobile"
}
```

### Step 3 — Booking confirmed
```json
{
  "event": "booking_confirmed",
  "lead_id": "ORTHO-8842",
  "clinic_location": "Indiranagar - Bengaluru",
  "booking_status": "confirmed"
}
```

### GTM Setup
1. **Trigger 1:** Custom Event trigger, Event name = `booking_step_complete` — catches both steps 1 and 2, differentiated downstream by the `step_number` variable.
2. **Trigger 2:** Custom Event trigger, Event name = `booking_confirmed` — kept separate because it doubles as the conversion event, not just a funnel step.
3. **Variables:** Data Layer Variables for `step_number`, `clinic_location`, `specialty`, `lead_id`, `device_type`, `booking_status` — pulled directly from each pushed object.
4. **Tags:** A GA4 Event tag fires on each trigger, mapping the Data Layer Variables into GA4 event parameters.

### Surfacing Drop-off in GA4
- Go to **Explore > Funnel Exploration**.
- Build the funnel: Step 1 = `booking_step_complete` where `step_number = 1` → Step 2 = `booking_step_complete` where `step_number = 2` → Step 3 = `booking_confirmed`.
- Turn on **"Show elapsed time"** to see how long users sit on step 2 before dropping — typically the biggest leak point, since it's the step asking for a phone number.
- Segment the funnel by `clinic_location` to check whether one specific clinic's form (e.g. a broken date-picker) is bleeding disproportionately more users than others.

---

## 3. Google Ads Conversion Import

**Import `booking_confirmed`, not `call_now_click` or `whatsapp_chat_open`.**

Both `call_now_click` and `whatsapp_chat_open` are ambiguous top-of-funnel signals — a phone tap could be a wrong number, idle curiosity, or someone just checking the clinic's hours. `booking_confirmed` is the only event backed by a fully completed booking flow with validated lead information and explicit confirmation — the closest digital proxy to a real lead actually reaching the clinic. Importing this as the Google Ads conversion action means the bidding algorithm optimizes spend toward people who look like they'll genuinely complete the form, not just people who tap a phone icon.
