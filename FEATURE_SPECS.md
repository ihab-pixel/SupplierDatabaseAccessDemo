# Feature Specification – Supplier / Services Search

## Overview

A modal overlay that lets users search and filter a supplier database, browse supplier profiles and their services, and select a service to populate the current proposal entry.

---

## 1. Opening the Modal

The modal can be triggered from two places:
- **Browse button (🔍)** next to the Supplier field in the Entry Edit Panel — opens with no pre-applied filters
- **Search icon (🔍)** in the Lodging row of the proposal grid — opens pre-filtered to **Category: Hotel**

---

## 2. Layout

- Full-screen overlay with a centred dialog (max 1060px wide, max 700px tall)
- **Left sidebar** — search input, filters, result count, clear button
- **Right panel** — scrollable list of supplier cards
- Clicking the background overlay closes the modal without selecting anything

---

## 3. Search

- Free-text input placeholder: "Search supplier name…"
- Matches against supplier name, case-insensitive, real-time (no submit required)
- Combined with all active filters (AND logic — see §6)

---

## 4. Filters

### 4.1 Supplier Filters

| Filter | Options |
|---|---|
| Location | All locations · Casablanca · Chefchaouen · Fes · Marrakech · Rabat · Agadir |
| Category | All categories · Hotel · Activity · Restaurant · Guide · Driver · Transportation · Other |
| Hotel Category | All · 5+ · 5 · 4 · 3 · 2 · 1 |
| Grade | All · A+ · A · A− · B+ · B · B− · C+ · C · C− |
| Spoken Languages | Any · English · French · Arabic · Spanish · Italian · German |
| Avg. Bookings / Month | Any · 5+ · 10+ · 20+ · 30+ |

### 4.2 Service Filters

| Filter | Options |
|---|---|
| Category | All · Accommodation · Experience · Guide · Transportation · Activity · Driver · Restaurant · Other |
| Sub-category | All · Accommodation · Cooking Class · Culture · Excursion · Food · Guide · Hot Air Balloon · Monument/Site · Museum · Nature / Animals · Photoshoot · Private Driving · Restaurant · Show · Trekking · Well-being · Wine Tasting · ATV |

### 4.3 Conditional Filter Visibility

- **Hotel Category** is shown only when the Category filter is set to **Hotel**; hidden and reset otherwise
- **Spoken Languages** is hidden and reset when the Category filter is set to **Hotel**; shown for all other categories

### 4.4 Additional Controls

- **Show unavailable** toggle — when off (default), hides suppliers that are disabled or temporarily closed; when on, includes them with status badges
- **Result count** — live label showing the number of suppliers matching current filters (e.g. "8 suppliers")
- **Clear all filters** button — resets all dropdowns to their default (All / Any), clears the search text; the Show unavailable toggle retains its current state

---

## 5. Supplier Cards

### 5.1 Collapsed State

Each card shows:

| Element | Detail |
|---|---|
| Name | Supplier display name |
| Status badge | `Disabled` (red) or `Temporarily closed` (orange) — shown only when applicable |
| Category badge(s) | Orange; one per assigned category (e.g. Hotel, Activity) |
| Grade badge | Blue; e.g. A+, B− |
| Location badge | Green with 📍 icon; city name |
| Hotel star badge | Yellow; e.g. 5+★, 4★ — shown for Hotel category only |
| Languages badge | Purple with 🗣 icon; e.g. English, French — shown for Guide category only |
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
- **Category** — matches if the supplier has *at least one* category matching the selected value
- **Hotel Category** — matches if the supplier's star rating equals the selected value
- **Grade** — exact match against the selected grade
- **Location** — exact match against the supplier's city
- **Spoken Languages** — matches if the supplier's language list includes *at least* the selected language
- **Avg. Bookings / Month** — matches if `avgBookingsPerMonth ≥ threshold` (e.g. selecting "10+" shows suppliers with 10 or more bookings per month)
- **Show unavailable off** — excludes any supplier where `disabled = true` OR `temporarilyClosed = true`
- **Search text** — matches if the supplier name contains the typed string (case-insensitive)
- Result count updates live after every filter or search change

---

## 7. Service Selection

1. User clicks **Select** on a service row
2. The following fields are auto-filled in the Entry Edit Panel: Supplier, Service, Unit Cost, Unit Quote
3. A toast notification appears: `✓ Entry saved` (3-second auto-dismiss)
4. The modal closes

---

## 8. Data Model

### Supplier

| Property | Type | Description |
|---|---|---|
| `name` | String | Display name |
| `location` | String | City |
| `categories` | String[] | One or more: Hotel, Activity, Restaurant, Guide, Driver, Transportation, Other |
| `grade` | String | Quality grade: A+ … C− |
| `hotelCategory` | String \| null | Star rating (Hotels only): 5+, 5, 4, 3, 2, 1 |
| `languages` | String[] | Spoken languages (non-Hotel suppliers) |
| `avgBookingsPerMonth` | Integer | Rolling average monthly booking volume |
| `approved` | Boolean | Whether the supplier is vetted |
| `disabled` | Boolean | Supplier is permanently unavailable |
| `temporarilyClosed` | Boolean | Supplier is temporarily unavailable |
| `services` | Service[] | List of offered services |

### Service

| Property | Type | Description |
|---|---|---|
| `name` | String | Service display name |
| `category` | String | Broad service category |
| `subCategory` | String | Specific service sub-type |
| `unitCost` | Number | Wholesale cost (USD) |
| `unitQuote` | Number | Client-facing price (USD) |
