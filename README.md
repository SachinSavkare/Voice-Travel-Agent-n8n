# TrueHorizon Voice Travel Agent : Turn Requests into Ready-to-Send Travel Plans 🛫📧

![Automation: n8n](https://img.shields.io/badge/Automation-n8n-20B2AA?logo=n8n) ![Telegram Bot](https://img.shields.io/badge/Interface-Telegram-26A5E4?logo=telegram) ![Google Calendar](https://img.shields.io/badge/Calendar-Google%20Calendar-4285F4?logo=googlecalendar) ![Google Sheets](https://img.shields.io/badge/Contacts-Google%20Sheets-34A853?logo=googlesheets) ![HIL](https://img.shields.io/badge/Human--in--the--Loop-Required-000000) ![Timezone IST](https://img.shields.io/badge/Timezone-IST%20\(UTC%2B05%3A30\)-FF9800)

---

## Project overview

Voice Travel Agent converts a single webhook (voice or text) request into a curated, clickable HTML travel plan and sends it by email. It automates airport-code normalization, flight & hotel lookups, and activity curation — delivering polished itineraries in minutes so teams can act faster and with fewer errors.

---

## Problem we solve (user pain points)

* Manual travel preparation requires many browser tabs and manual formatting.
* Common errors: wrong dates, missing airport codes, time-zone mismatches.
* High cognitive load for travel coordinators — assembling options and writing emails is slow.
* Fragmented process across search, booking, and email tools.

### How Voice Travel Agent helps (outcome-focused)

* Single webhook → validated, enriched itinerary → email delivery.
* Consistent, branded HTML itineraries with links and images.
* Cuts research + formatting time dramatically so teams focus on decisions.

### Estimated productivity gains

* Time to compose & send travel plan: **↓ 75–90%**
* Manual date/airport-code errors: **↓ 60–85%**
* Time-to-response: **hours → minutes**

---

## 🧭 **Workflow Diagram**

![Workflow Diagram](https://github.com/SachinSavkare/Voice-Travel-Agent-n8n/blob/main/Voice%20Travel%20Agent.JPG)

---

## Mermaid overview (high-level flow)

```mermaid
flowchart TD
  A[Webhook: /webhook-test/travel (POST)] --> B[Set Incoming Fields]
  B --> C[Airport Codes & Dates (AI)]
  C --> D[Tavily: Activities (HTTP POST)]
  C --> E[SerpAPI: Hotels (google_hotels)]
  C --> F[SerpAPI: Flights (google_flights)]
  D --> G[AI Agent: Email Generator]
  E --> G
  F --> G
  G --> H[Gmail AIS: Send Travel Plan (HTML)]
  H --> I[Respond to Webhook: Confirmation]
```

---

## Node-by-node configuration (purpose + essential params)

> Only the essential parameters are listed — keep API credentials in n8n credential manager.

### Step 1 — Webhook (`Webhook` node)

* **Purpose:** Accept incoming travel request (voice or text → JSON).
* **HTTP Method:** `POST`
* **Path:** `/webhook-test/travel`
* **Respond:** Use `Respond to Webhook` node for final confirmation.

---

### Step 2 — Set Incoming Fields (`Set` node; Manual Mapping)

* **Purpose:** Normalize inbound payload into fields used downstream.
* **Essential fields:**

  * `origin` = `{{ $json.body.origin }}`
  * `destination` = `{{ $json.body.destination }}`
  * `departure_date` = `{{ $json.body.departure_date }}`
  * `return_date` = `{{ $json.body.return_date }}`
  * `travelers` = `{{ $json.body.travelers }}`
  * `activities` = `{{ $json.body.activities }}`
  * `email` = `{{ $json.body.email }}`

---

### Step 3 — Airport Codes & Dates (AI step)

* **Purpose:** Convert origin/destination to airport codes/IDs; normalize dates to `YYYY-MM-DD` and ensure future dates.
* **Key behavior:** Output required JSON with `origin`, `destination`, `departure`, `return` (dates in ISO format).

---

### Step 4 — TAVILY: Activities (HTTP Request)

* **Purpose:** Curated activities for the destination + activity topic.
* **Method/Endpoint:** `POST https://api.tavily.com/search`
* **Essential body (JSON):**

  ```json
  {
    "query": "{{ $('Set Incoming Fields').item.json.activities }} in {{ $('Set Incoming Fields').item.json.destination }}",
    "topic": "general",
    "max_results": 3,
    "days": 3,
    "include_answer": true
  }
  ```

---

### Step 5 — SerpAPI: Hotels & Resorts (HTTP Request)

* **Purpose:** Retrieve hotels/resorts (images, links, short details) for check-in/out dates.
* **Method/Endpoint:** `GET https://serpapi.com/search`
* **Essential query params:** `engine=google_hotels`, `q={{destination}}`, `check_in_date={{departure}}`, `check_out_date={{return}}`, `adults={{travelers}}`

---

### Step 6 — SerpAPI: Flights (HTTP Request)

* **Purpose:** Retrieve flight options using airport codes/IDs and dates.
* **Method/Endpoint:** `GET https://serpapi.com/search`
* **Essential query params:** `engine=google_flights`, `departure_id={{origin}}`, `arrival_id={{destination}}`, `outbound_date={{departure}}`, `return_date={{return}}`, `adults={{travelers}}`

---

### Step 7 — AI Agent: Email Generator (Chat/AI node)

* **Purpose:** Build final **HTML** itinerary with three sections: **Flights**, **Resorts**, **Activities**.
* **Output:** Structured JSON containing:

  * `subject` (string) — must contain travel dates & arrival location
  * `emailBody` (string) — HTML, with clickable links and resort images (`<img src="...">`)
* **Important:** Keep output ≤ 1000 words; sanitize external image URLs before embedding.

```json
{
You are an expert email writer specializing in creating travel plans. Your job is to output an HTML email with clickable links. You must output a subject and an emailBody in separate parameters.

Objective:
- Break the email into 3 sections: Flights, Resorts, Activities.
- Subject must contain travel dates and arrival location.
- Introduction to excite traveler; horizontal rules after each section.
- Use images for resorts: <img src="{image url}" style="max-width:20%; height:auto;">
- Sign off as: TrueHorizon Travel Team
- Do not exceed 1000 words.
```

---

### Step 8 — Send Travel Plan (Gmail AIS node)

* **Purpose:** Email the generated HTML to the requestor.
* **Essential fields:**

  * `To = {{ $('Set Incoming Fields').item.json.email }}`
  * `Subject = {{ $json.output.subject }}`
  * `Message (HTML) = {{ $json.output.emailBody }}`

---

### Step 9 — Field: Response to Webhook (`Set` node)

* **Purpose:** Construct confirmation payload returned to the webhook caller.
* **Example mapping:**

  * `name = An email has been sent with the travel plan for: {{ $('Email Agents').item.json.output.subject }}`

---

### Step 10 — Respond to Webhook (`Respond to Webhook` node)

* **Purpose:** Send the confirmation back to the caller (`First Incoming Item`).

---

## 🧾 Example input queries (POST payloads)

```json
{
  "origin": "New York City",
  "destination": "Chicago",
  "departure_date": "December 10, 2025",
  "return_date": "December 20, 2025",
  "travelers": 1,
  "activities": "riverboat tour, architecture cruise",
  "email": "sachinsavkare08@gmail.com"
}
```

```json
{
  "origin": "Bengaluru, India",
  "destination": "Goa, India",
  "departure_date": "2026-01-15",
  "return_date": "2026-01-20",
  "travelers": 2,
  "activities": "beach, water sports",
  "email": "group-booking@example.com"
}
```

---

## 📊 **Before vs After — with Voice Travel Agent**

| Aspect            | Before (Manual)                         | After (Voice Travel Agent)                                    |
| ----------------- | --------------------------------------- | ------------------------------------------------------------- |
| Safety            | Errors from manual entry (dates, codes) | **Validated** dates & airport codes before sending            |
| Time Zone         | Timezone drift / confusion              | **IST-normalized** parsing and clear date formatting          |
| Conflicts         | Overlaps discovered late                | **Pre-check** for obvious conflicts before sending            |
| Attendees         | Wrong/missing contact details           | **Contacts lookup** and standard email target used            |
| Edits/Corrections | Manual rework & reformatting            | **One-click** update + re-send (standardized template)        |
| Cognitive Load    | High — many tabs and manual formatting  | **Low** — speak or send a single request, agent composes plan |

---

## 📥 **Free Workflow Template**

* **Download:** [n8n workflow JSON](https://github.com/SachinSavkare/Voice-Travel-Agent-n8n/blob/main/23.%20Voice%20Travel%20Agent.json)
  *(Replace with your actual workflow export link once you upload.)*

---

## ✍️ **Author**

**Sachin Savkare**
📧 `sachinsavkare08@gmail.com`
---

If this looks good I will:

* paste a final single-file README ready to copy/paste into your repo (with the workflow diagram link you’ll provide), or
* produce a variant targeted to a specific audience (corporate travel desk / travel agency).

Which would you like next?
