# Feature Specifications – Supplier Database Access Demo

## Overview

A web-based travel proposal editor that lets trip designers build day-by-day itineraries by browsing a supplier database, selecting services, and assembling them into a structured proposal document.

---

## 1. Top Bar

### 1.1 Proposal Metadata

| Field | Type | Behaviour |
|---|---|---|
| Proposal Name | Text input | Editable inline; persists during session |
| Start Date | Text input | Free-text date entry (e.g. `08-Sept-2026`) |
| Nights | Integer counter | Incremented / decremented with `+` / `−` buttons; minimum 1 |
| End Date | Read-only text | Auto-calculated from Start Date + Nights |
| PAX | Number input | Passenger count; minimum 1 |

### 1.2 Display Toggles

- **Breakdown** – show/hide per-line cost breakdown in the proposal grid
- **Hide Dates** – suppress date labels from the printed/exported output

### 1.3 Action Buttons

| Button | Action |
|---|---|
| Itinerary | Navigate to / generate itinerary view |
| Pricing | Navigate to / generate pricing view |
| Save | Persist all proposal changes |
| Close | Exit the proposal editor |

---

## 2. Proposal Grid

### 2.1 Layout

- Horizontally scrollable day grid
- Each **day column** shows: weekday name, date, month, and destination location
- **Row types** per day:
  - Morning
  - Afternoon
  - Evening
  - Driving (travel duration between destinations)
  - Lodging (hotel search field)

### 2.2 Activity Entry Cards

**Display**
- Supplier name badge (orange tag)
- Service name
- Hover reveals: Copy (⧉) and Delete (🗑) action buttons

**Interactions**
- Click a card → opens the Entry Edit Panel (§3)
- Click Copy → duplicates entry to clipboard / adjacent slot
- Click Delete → removes entry from the grid slot

**"New Entry…" Placeholder**
- Below each activity card in a slot; clicking it creates a blank entry and opens the Entry Edit Panel

### 2.3 Lodging Row

- Text input per day for hotel name
- Search icon (🔍) → opens Supplier Modal pre-filtered to **Hotel** category
- Selecting a hotel auto-fills the lodging field for that day

### 2.4 Driving Row

- Displays estimated drive time between consecutive locations (e.g. `4h`, `N/A`)
- Copy (⧉) button per cell copies the duration text

---

## 3. Entry Edit Panel

Slides in from the right when an activity card is selected or created.

### 3.1 Fields

| Field | Type | Notes |
|---|---|---|
| Included in price | Checkbox | Checked by default |
| Supplier | Text + Browse (🔍) | Browse opens Supplier Modal |
| Service | Text (read-only) | Auto-filled on supplier service selection |
| Unit Cost | Number | Wholesale cost in USD |
| Unit Quote | Number | Client-facing price in USD |
| Notes to Ops | Textarea | Internal operational notes; not exported to client |
| Start Time | Time (HH:MM) | Optional scheduled start time |
| Do not generate in proposal | Checkbox | Hides entry from exported proposal output |

### 3.2 Actions

- **Save** – writes field values back to the entry card and closes the panel
- **Cancel** – discards changes and closes the panel

---

## 4. Supplier Modal

Large overlay dialog for browsing and selecting suppliers.

### 4.1 Layout

- **Left sidebar** – search bar, filters, result count
- **Right area** – scrollable list of supplier cards

### 4.2 Search

- Free-text input: matches against supplier name (case-insensitive, real-time)

### 4.3 Filters

#### Supplier Filters

| Filter | Options |
|---|---|
| Location | All locations, Casablanca, Chefchaouen, Fes, Marrakech, Rabat, Agadir |
| Category | All categories, Hotel, Activity, Restaurant, Guide, Driver, Transportation, Other |
| Hotel Category *(Hotel only)* | All, 5+, 5, 4, 3, 2, 1 |
| Grade | All, A+, A, A−, B+, B, B−, C+, C, C− |
| Spoken Languages *(non-Hotel)* | Any, English, French, Arabic, Spanish, Italian, German |
| Avg. Bookings / Month | Any, 5+, 10+, 20+, 30+ |

#### Service Filters

| Filter | Options |
|---|---|
| Category | All, Accommodation, Experience, Guide, Transportation, Activity, Driver, Restaurant, Other |
| Sub-category | All, Accommodation, Cooking Class, Culture, Excursion, Food, Guide, Hot Air Balloon, Monument/Site, Museum, Nature / Animals, Photoshoot, Private Driving, Restaurant, Show, Trekking, Well-being, Wine Tasting, ATV |

#### Additional Controls

- **Show unavailable** toggle – when off, hides disabled / temporarily closed suppliers
- **Result count** label – displays number of suppliers matching current filters
- **Clear all filters** button – resets all filters and search text to defaults

### 4.4 Supplier Cards

**Collapsed state** displays:

| Element | Description |
|---|---|
| Name | Supplier display name |
| Status badge | `Disabled` (red) or `Temporarily closed` (orange) if applicable |
| Category badge(s) | Orange; e.g. Hotel, Activity |
| Grade badge | Blue; e.g. A+, B− |
| Location badge | Green with 📍; city name |
| Hotel star badge | Yellow; e.g. 5+★, 4★ *(Hotels only)* |
| Languages badge | Purple with 🗣; e.g. English, French *(Guides only)* |
| Service count | Number of services offered |
| Chevron | ▼ collapsed / △ expanded |

**Expanded state** (click to toggle) – shows service table:

| Column | Description |
|---|---|
| Service Name | Name with secondary sub-category label |
| Unit Cost | Wholesale cost in USD (gray) |
| Unit Quote | Client-facing price in USD (blue, bold) |
| Select | Orange button – selects this service into the open entry |

### 4.5 Service Selection

Clicking **Select** on a service:
1. Auto-fills **Supplier**, **Service**, **Unit Cost**, and **Unit Quote** in the Entry Edit Panel
2. Shows a toast confirmation: `✓ Entry saved`
3. Closes the modal

### 4.6 Modal Dismissal

- Click the background overlay to close without selecting

---

## 5. Supplier Data Model

Each supplier record contains:

| Property | Type | Description |
|---|---|---|
| `name` | String | Display name |
| `location` | String | City |
| `categories` | String[] | One or more: Hotel, Activity, Restaurant, Guide, Driver, Transportation |
| `grade` | String | Quality grade (A+ … C−) |
| `hotelCategory` | String/null | Star rating string (Hotels only) |
| `languages` | String[] | Spoken languages (Guides / non-Hotels) |
| `avgBookingsPerMonth` | Integer | Rolling average monthly booking volume |
| `approved` | Boolean | Whether supplier is vetted |
| `disabled` | Boolean | Supplier is permanently unavailable |
| `temporarilyClosed` | Boolean | Supplier is temporarily unavailable |
| `services` | Service[] | List of offered services |

Each **Service** contains:

| Property | Type | Description |
|---|---|---|
| `name` | String | Service display name |
| `category` | String | Broad service category |
| `subCategory` | String | Specific service sub-type |
| `unitCost` | Number | Wholesale cost (USD) |
| `unitQuote` | Number | Client-facing price (USD) |

---

## 6. Filter Logic

- All active filters are **AND-ed** together (a supplier must satisfy every active filter to appear)
- The **Category** filter matches if the supplier has *at least one* matching category
- The **Avg. Bookings / Month** filter matches suppliers whose `avgBookingsPerMonth` is **≥** the selected threshold
- The **Spoken Languages** filter matches if the supplier supports *at least* the selected language
- **Hotel Category** and **Languages** filters are hidden (and reset) when they do not apply to the selected Category
- The **Show unavailable** toggle, when off, excludes suppliers where `disabled === true` OR `temporarilyClosed === true`
- **Clear all filters** resets every dropdown to its "All / Any" default, clears the search text, and shows unavailable suppliers per the current toggle state

---

## 7. Notifications

- **Toast** – floating message centred at the bottom of the screen
  - Appears for 3 seconds then auto-dismisses
  - Used to confirm: service selection, entry save

---

## 8. Supplier Catalogue (Current Mock Data)

| # | Name | Location | Categories | Grade | Stars | Avg Bookings/mo |
|---|---|---|---|---|---|---|
| 1 | La Mamounia | Marrakech | Hotel | A+ | 5+ | 38 |
| 2 | Four Seasons Marrakech | Marrakech | Hotel | A+ | 5+ | 31 |
| 3 | Royal Mansour Marrakech | Marrakech | Hotel | A+ | 5+ | 24 |
| 4 | Sofitel Casablanca Tour Blanche | Casablanca | Hotel | A− | 5 | 17 |
| 5 | Palais Faraj | Fes | Hotel | A | 5 | 13 |
| 6 | Riad Fes – Relais & Châteaux | Fes | Hotel | A | 5 | 21 |
| 7 | Dar Rhizlane | Marrakech | Hotel | A− | 4 | 9 |
| 8 | La Maison Arabe | Marrakech | Hotel, Restaurant | A | 5 | 16 |
| 9 | Riad 72 *(Disabled)* | Marrakech | Hotel | B+ | 4 | 4 |
| 10 | Rick's Cafe | Casablanca | Restaurant | A | — | 22 |
| 11 | Dar Moha | Marrakech | Restaurant | A+ | — | 28 |
| 12 | Atlas Luxury Tours | Casablanca | Driver, Transportation | A− | — | 33 |
| 13 | Mohammed El Fassi – Licensed Guide | Fes | Guide | A | — | 7 |
| 14 | Chefchaouen Blue Escapes | Chefchaouen | Guide, Activity | B+ | — | 11 |
| 15 | Maroc Désert Expériences | Marrakech | Activity | B+ | — | 19 |

---

## 9. Out-of-Scope / Placeholder Features

The following UI elements are present but not yet functional:

- Day header navigation buttons (◀ ▶ ↵ ↤)
- **Itinerary** and **Pricing** action buttons (no target view implemented)
- **Save** button (no backend persistence)
- **Breakdown** and **Hide Dates** toggles (no rendering effect)
