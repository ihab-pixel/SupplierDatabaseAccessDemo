# Feature Specification – Salesforce × WhatsApp Daily Activity Digest

## Overview

Each morning, every traveller on an active trip receives a WhatsApp message summarising their day's activities. The message is assembled from confirmed Salesforce booking records and sent automatically via the WhatsApp Business API through Salesforce Flow or Apex.

---

## 1. Goals

| Goal | Notes |
|---|---|
| Keep travellers informed without app downloads | WhatsApp has near-universal adoption |
| Reduce inbound support calls | Proactive information delivery |
| Surface operational details at the right moment | Sent the morning of the relevant day |
| Allow the ops team to add ad-hoc notes | "Important Info" field on each activity record |

---

## 2. Salesforce Data Model

### 2.1 Objects Involved

| Object | Role |
|---|---|
| `Contact` | The traveller; holds WhatsApp number |
| `Trip__c` | One record per booking; links Contact → itinerary dates |
| `TripDay__c` | One record per calendar day within a trip |
| `Activity__c` | One entry-level activity in a day (hotel, guide, excursion, etc.) |
| `Supplier__c` | The service provider |
| `Guide__c` | Optional; guide assigned to the activity |

### 2.2 Key Fields

#### Contact
| Field | API Name | Type | Notes |
|---|---|---|---|
| WhatsApp Number | `WhatsApp_Number__c` | Phone | E.164 format, e.g. `+212600000000` |
| WhatsApp Opt-In | `WhatsApp_Opt_In__c` | Checkbox | Must be `true` to receive messages |
| Preferred Language | `Preferred_Language__c` | Picklist | `en`, `fr`, `ar` — drives message locale |

#### Trip__c
| Field | API Name | Type | Notes |
|---|---|---|---|
| Contact | `Contact__c` | Lookup | The traveller |
| Start Date | `Start_Date__c` | Date | First day of trip |
| End Date | `End_Date__c` | Date | Last day of trip |
| Status | `Status__c` | Picklist | Only `Confirmed` trips trigger messages |

#### TripDay__c
| Field | API Name | Type | Notes |
|---|---|---|---|
| Trip | `Trip__c` | Master-Detail | Parent trip |
| Date | `Date__c` | Date | Calendar date of this day |
| Location | `Location__c` | Text | City/region for the day |
| Day Number | `Day_Number__c` | Number | e.g. Day 3 of 7 |

#### Activity__c
| Field | API Name | Type | Notes |
|---|---|---|---|
| Trip Day | `Trip_Day__c` | Master-Detail | Parent day |
| Supplier | `Supplier__c` | Lookup | Service provider |
| Guide | `Guide__c` | Lookup | Optional; assigned guide |
| Activity Name | `Name` | Text | Display name of the activity |
| Category | `Category__c` | Picklist | Hotel · Restaurant · Activity · Guide · Transfer · Other |
| Start Time | `Start_Time__c` | Time | Local time, e.g. `09:00` |
| End Time | `End_Time__c` | Time | Optional |
| Ticket Required | `Ticket_Required__c` | Checkbox | Triggers ticket reminder line |
| Ticket Reference | `Ticket_Reference__c` | Text | Booking/voucher code |
| Important Info | `Important_Info__c` | Long Text Area | Freeform ops note, e.g. "Wear closed shoes, no sandals" |
| Include in Message | `Include_In_Message__c` | Checkbox | Default `true`; uncheck to suppress from digest |
| Sort Order | `Sort_Order__c` | Number | Manual ordering within the day |

#### Guide__c
| Field | API Name | Type | Notes |
|---|---|---|---|
| Full Name | `Name` | Text | |
| Mobile | `Mobile__c` | Phone | Displayed in message |
| Languages | `Languages__c` | Text | e.g. "English, French" |

---

## 3. Message Format

Messages use a WhatsApp-approved template (`daily_activity_digest`). Templates are submitted to Meta for approval before go-live.

### 3.1 Template Structure

```
*Good morning, {{1}}!* 🌅

Here is your programme for *{{2}}* (Day {{3}} of {{4}}):

{{5}}

Have a wonderful day! 🇲🇦
_Reply STOP to opt out._
```

Where:
- `{{1}}` — traveller first name
- `{{2}}` — location (from `TripDay__c.Location__c`)
- `{{3}}` — day number
- `{{4}}` — total trip days
- `{{5}}` — activity block (see §3.2)

### 3.2 Activity Block

Each activity on the day generates one block. Blocks are joined with a blank line.

**Full block (all fields populated):**
```
🕘 *09:00* — Medina Walking Tour
   👤 Guide: Mohammed El Fassi · 📞 +212 6XX XXX XXX
   🎟 Ticket: VCH-2024-0312
   ⚠️ Wear closed shoes — no sandals allowed inside mosques.
```

**Minimal block (no guide, no ticket, no important info):**
```
🕘 *14:00* — Dinner at Dar Moha
```

**Transfer block:**
```
🚐 *08:30* — Transfer: Marrakech → Essaouira
   👤 Driver: Hassan Benali · 📞 +212 6XX XXX XXX
```

### 3.3 Icon Legend

| Icon | Used for |
|---|---|
| 🕘 | Standard activity |
| 🏨 | Hotel check-in / check-out |
| 🍽 | Restaurant / meal |
| 🚐 | Transfer / transport |
| 🧭 | Guided excursion |
| 👤 | Guide or driver name |
| 📞 | Contact phone number |
| 🎟 | Ticket / voucher reference |
| ⚠️ | Important info / warning |

Category → icon mapping is a formula field `Message_Icon__c` on `Activity__c`.

---

## 4. Sending Logic

### 4.1 Trigger

A **Salesforce Scheduled Flow** runs daily at **07:00 local time** (configurable per trip timezone). It:

1. Queries all `TripDay__c` records where `Date__c = TODAY()`
2. Joins to parent `Trip__c` where `Status__c = 'Confirmed'`
3. Joins to `Contact` where `WhatsApp_Opt_In__c = true` and `WhatsApp_Number__c != null`
4. For each qualifying trip day, calls the **WhatsApp Send Action** (see §5)

### 4.2 Activity Ordering

Activities are ordered by:
1. `Sort_Order__c` ASC (manual priority)
2. `Start_Time__c` ASC (chronological fallback)

Activities with `Include_In_Message__c = false` are excluded entirely.

### 4.3 Empty Days

If a `TripDay__c` has zero qualifying activities, no message is sent for that day. The flow logs a `MessageLog__c` record with status `Skipped – No Activities`.

### 4.4 Multi-Traveller Trips

If a trip has multiple travellers (modelled as separate `Contact` records linked to the same `Trip__c`), each traveller receives their own personalised message.

---

## 5. WhatsApp Integration (Salesforce Side)

### 5.1 Recommended Approach

Use **Salesforce Messaging for WhatsApp** (Digital Engagement / Unified Messaging) or a certified third-party connector (e.g. 360dialog, Twilio, Bird) depending on existing org licensing.

| Approach | Pros | Cons |
|---|---|---|
| Native Salesforce Messaging | Tight CRM integration, conversation history in org | Requires Digital Engagement licence |
| 360dialog / Twilio connector | Lower cost, simpler setup | Separate platform to manage |

The interface below is connector-agnostic; the send action is wrapped in an Apex `HttpCalloutMock`-testable class.

### 5.2 Apex Send Action

```apex
public class WhatsAppDailyDigestAction {

    @InvocableMethod(label='Send Daily Activity Digest')
    public static void send(List<DigestRequest> requests) {
        for (DigestRequest req : requests) {
            String body = buildMessageBody(req);
            WhatsAppAPIService.sendTemplateMessage(
                req.whatsappNumber,
                'daily_activity_digest',
                req.languageCode,
                body,
                req.tripDayId
            );
        }
    }

    public class DigestRequest {
        @InvocableVariable(required=true) public String whatsappNumber;
        @InvocableVariable(required=true) public String languageCode;
        @InvocableVariable(required=true) public Id tripDayId;
        @InvocableVariable(required=true) public String travelerFirstName;
        @InvocableVariable(required=true) public String location;
        @InvocableVariable(required=true) public Integer dayNumber;
        @InvocableVariable(required=true) public Integer totalDays;
        @InvocableVariable(required=true) public List<Activity__c> activities;
    }
}
```

### 5.3 Message Log

Every send attempt (success or failure) creates a `MessageLog__c` record:

| Field | Type | Notes |
|---|---|---|
| `Trip_Day__c` | Lookup | Day the message covers |
| `Contact__c` | Lookup | Recipient |
| `Sent_At__c` | DateTime | Timestamp of API call |
| `Status__c` | Picklist | `Sent` · `Failed` · `Skipped – No Activities` · `Skipped – Opt Out` |
| `Error_Message__c` | Long Text | API error body if failed |
| `Message_Preview__c` | Long Text | Truncated copy of the message sent (for auditing) |

---

## 6. Opt-In / Opt-Out

| Event | Mechanism |
|---|---|
| Opt-in | Traveller sends any message to the WhatsApp Business number **or** ops team checks `WhatsApp_Opt_In__c` manually |
| Opt-out | Traveller replies `STOP` → webhook sets `WhatsApp_Opt_In__c = false` automatically |
| Re-opt-in | Traveller replies `START` → webhook sets `WhatsApp_Opt_In__c = true` |

All opt-out/in changes are tracked in `Contact` history (field audit trail).

---

## 7. Localisation

| Language | Code | Notes |
|---|---|---|
| English | `en` | Default |
| French | `fr` | Common for European travellers |
| Arabic | `ar` | Right-to-left; template must be approved separately by Meta |

Separate WhatsApp templates must be submitted for each language. Template names follow the convention `daily_activity_digest_[lang]`.

---

## 8. Ops Team Interface (Salesforce UI)

### 8.1 Quick Actions on TripDay__c

- **Preview Message** — Lightning quick action that renders the message as it will appear on WhatsApp (read-only modal)
- **Send Now** — Manual trigger for a specific day (overrides the 07:00 schedule); requires `Send_WhatsApp_Message` custom permission

### 8.2 Activity__c List View on TripDay__c

The related list shows activities in send order with columns:

| Column | Field |
|---|---|
| Sort | `Sort_Order__c` |
| Icon | `Message_Icon__c` |
| Time | `Start_Time__c` |
| Activity | `Name` |
| Guide | `Guide__r.Name` |
| Ticket | `Ticket_Reference__c` |
| Important Info | `Important_Info__c` (truncated) |
| In Message? | `Include_In_Message__c` |

Drag-to-reorder on `Sort_Order__c` is a stretch goal (requires custom LWC).

### 8.3 Bulk Suppress

Checkbox on `Activity__c` list view + mass-update action to flip `Include_In_Message__c` for multiple records at once.

---

## 9. Error Handling & Retry

| Scenario | Behaviour |
|---|---|
| API timeout / 5xx from WhatsApp | Retry up to 3 times with exponential backoff (2 s, 4 s, 8 s); log `Failed` after exhaustion |
| Invalid phone number | Log `Failed`, flag `Contact.WhatsApp_Number_Invalid__c = true`, send internal alert to ops |
| Template not approved | Block send; surface validation error in Salesforce UI; log `Failed` |
| Daily flow failure (unhandled exception) | Platform email alert to integration admin; manual re-send via Quick Action |

---

## 10. Compliance & Privacy

- WhatsApp Business Policy requires explicit opt-in for outbound template messages — enforced by `WhatsApp_Opt_In__c` gate in §4.1.
- Phone numbers are masked in logs after 90 days (GDPR-aligned data retention policy).
- `Message_Preview__c` on `MessageLog__c` is redacted of personal details after 90 days.
- Only users with the `View_Traveler_PII` permission set can see raw phone numbers.

---

## 11. Acceptance Criteria

| # | Criteria |
|---|---|
| AC-1 | A traveller with `WhatsApp_Opt_In__c = true` receives exactly one message per trip day by 07:15 local time |
| AC-2 | Activities are listed in `Sort_Order__c` then `Start_Time__c` order |
| AC-3 | Activities with `Include_In_Message__c = false` do not appear in the message |
| AC-4 | Guide name and phone appear if and only if a `Guide__c` is linked to the activity |
| AC-5 | Ticket reference appears if and only if `Ticket_Required__c = true` and `Ticket_Reference__c` is non-empty |
| AC-6 | Important Info appears if and only if `Important_Info__c` is non-empty |
| AC-7 | No message is sent on days with zero qualifying activities; a `Skipped` log is created |
| AC-8 | Replying `STOP` sets `WhatsApp_Opt_In__c = false` within 60 seconds |
| AC-9 | A failed API call is retried up to 3 times; outcome is always logged |
| AC-10 | Ops can preview and manually trigger a message from the `TripDay__c` record page |

---

## 12. Open Questions

| # | Question | Owner | Status |
|---|---|---|---|
| Q-1 | Which WhatsApp connector is licensed? (Native / 360dialog / Twilio / Bird) | IT / RevOps | Open |
| Q-2 | What timezone logic applies for multi-destination trips? (Per-day city timezone vs. traveller home timezone) | Ops | Open |
| Q-3 | Should guides receive a copy of the day's message as well? | Ops | Open |
| Q-4 | Arabic template — is right-to-left rendering acceptable within the WhatsApp client? | Product | Open |
| Q-5 | Is a trip ever shared by travellers who speak different languages? If so, send separate messages per language preference | Ops | Open |
| Q-6 | What is the cutoff time for ops to make same-day edits before the 07:00 send? | Ops | Open |

---

## 13. Out of Scope (v1)

- Two-way conversation handling (replies other than STOP/START)
- Rich media (images, PDFs, maps)
- Traveller self-service rescheduling via WhatsApp
- Push notifications to a native app
- SMS fallback for travellers without WhatsApp
