# Feature Specification – Supplier / Services Search

## Overview

A modal overlay that lets users search and filter a supplier database, browse supplier profiles and their services, and select a service to populate the current proposal entry.

---

## 1. Opening the Modal

The modal can be triggered from two places:
- **Browse button (🔍)** next to the Supplier field in the Entry Edit Panel — opens with no pre-applied filters
- **Search icon (🔍)** in the Lodging row of the proposal grid — opens with no pre-applied filters

---

## 2. Layout

- Full-screen overlay with a centred dialog (max 1060px wide, max 700px tall)
- **Left sidebar** — search input, filters, result count
- **Right panel** — scrollable list of supplier cards
- Clicking the background overlay closes the modal without selecting anything

---

## 3. Search

- Free-text input placeholder: "Search suppliers and services…"
- Searches across: **Supplier Name**, **Supplier Description**, **Service Name**, **Service Description**
- Multi-word input: all words must appear somewhere across those four fields (AND logic per word, OR logic across fields)
- Case-insensitive, real-time (no submit required)
- Combined with all active filters (AND logic — see §6)

---

## 4. Filters

### 4.1 Supplier Filters

| Filter | Options |
|---|---|
| Location | All (dynamically populated from system) |
| Hotel Category | All (dynamically populated from system) |
| Grade | All (dynamically populated from system) |
| Avg. Bookings / Month | All · 5+ · 10+ · 20+ · 30+ |

> **Avg. Bookings / Month** is calculated at supplier level: count of bookings with status `confirmed` created in the last 90 days, divided by 3.

### 4.2 Service Filters

| Filter | Options |
|---|---|
| Category | All · Accommodation · Experience · Guide · Transportation · Activity · Driver · Restaurant · Other |
| Sub-category | All · Accommodation · Cooking Class · Culture · Excursion · Food · Guide · Hot Air Balloon · Monument/Site · Museum · Nature / Animals · Photoshoot · Private Driving · Restaurant · Show · Trekking · Well-being · Wine Tasting · ATV |

---

## 5. Supplier Cards

### 5.1 Collapsed State

Each card shows:

| Element | Detail |
|---|---|
| Name | Supplier display name |
| Category badge(s) | Orange; one per assigned category (e.g. Hotel, Activity) |
| Grade badge | Blue; e.g. A+, B− |
| Location badge | Green with 📍 icon; city name |
| Hotel star badge | Yellow; e.g. 5+★, 4★ — shown for Hotel suppliers only |
| Languages badge | Purple with 🗣 icon; e.g. English, French — shown for Guide suppliers only |
| Avg bookings badge | Green; e.g. ~38 bkg/mo |
| Service count | Number of services offered |
| Chevron | ▼ when collapsed, △ when expanded |

### 5.2 Expanded State

Clicking a card header toggles the service table open/closed.

The service table has four columns:

| Column | Detail |
|---|---|
| Service Name | Primary name with secondary sub-category label below |
| Unit Cost | Wholesale cost in USD (gray text) |
| Unit Quote | Client-facing price in USD (blue, bold) |
| Select | Orange button — triggers service selection (see §6) |

---

## 6. Filter Logic

- All active filters and the search term are **AND-ed**: a supplier must satisfy every condition to appear
- **Hotel Category** — matches if the supplier's star rating equals the selected value
- **Grade** — exact match against the selected grade
- **Location** — exact match against the supplier's city
- **Avg. Bookings / Month** — matches if the computed avg ≥ threshold (e.g. "10+" shows suppliers with ≥ 10 avg bookings/month)
- **Search text** — all typed words must appear across [Supplier Name, Supplier Description, Service Name, Service Description] (case-insensitive); partial matches count
- Disabled and temporarily closed suppliers are always excluded
- Result count updates live after every filter or search change

---

## 7. Service Selection

1. User clicks **Select** on a service row
2. The following fields are auto-filled in the Entry Edit Panel: **Supplier**, **Service**, **Unit Cost**, **Unit Quote**
3. A toast notification appears: `✓ "[Service]" selected · fields autofilled` (3-second auto-dismiss)
4. The modal closes
