# Task 03 — Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

## 1. End-to-End Architecture

I would route the consultation form submission through a **direct API integration using a small backend service (Node.js serverless function)** rather than a no-code tool like Zapier or Make. The reason isn't speed — Make can usually still operate within the 2-minute SLA. The real reason is **control at scale**: a 9-clinic chain running continuous paid campaigns will eventually need custom logic such as retry queues, audit logs, and custom deduplication rules — all of which are difficult to manage in no-code tools once lead volume grows. A backend middleware also avoids exposing private HubSpot or Karix API keys in client-side JavaScript.

The flow: the frontend validates the form and pushes `consultation_form_submitted` into the dataLayer, consistent with Tasks 1 and 2. Since the form intentionally keeps friction low with only Name and Phone, **Clinic Preference isn't collected directly** — it is derived from campaign targeting (for example, a Bengaluru-specific Google Ads campaign maps the lead to the Bengaluru clinic). Before syncing, the backend normalizes the phone number, stripping spaces, symbols, and country codes like +91.

Using HubSpot's Contacts API, the backend searches by normalized phone; if a match exists, it updates the record, otherwise creates one with Name, Phone, Clinic Preference, Source = "Google Ads – Consultation Landing Page," and Lead Status = "New Enquiry."

Once CRM sync succeeds, two actions run asynchronously:
- Karix's WhatsApp Business API sends a pre-approved enquiry confirmation template to the patient.
- The conversion flows through GTM into GA4 and is imported into Google Ads.

## 2. Biggest Failure Point

> **HubSpot's default deduplication works on email, not phone, and this flow never collects an email** — relying on that default would create duplicate contacts for repeat submissions, especially risky since family members often share one number in Indian healthcare.

Fix: always search by normalized phone before creating a contact, and flag name mismatches for manual review instead of overwriting. A secondary risk is HubSpot API downtime or rate limits — every lead is first written to a temporary queue and retried automatically, so nothing is silently lost.

## 3. Protecting the 2-Minute WhatsApp SLA

Main risks include Karix rate limits during traffic spikes, queue backlog, or a WhatsApp template lapsing out of Meta approval.

I would log timestamps at each step:
- Form submitted  
- HubSpot synced  
- WhatsApp sent  

This allows total processing time to be tracked accurately.

If the delay exceeds **90 seconds**, an alert should trigger, giving enough buffer before the 2-minute SLA is breached.

Failed sends should retry automatically, and Karix's delivery dashboard should be reviewed daily to catch silent failures.

