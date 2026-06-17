---
name: contact-sync
description: Sync calendar meetings into Sophia's Notion Contacts database — create new records, enrich existing ones, and resolve duplicates interactively. Use this whenever Sophia wants to update her contacts from her calendar, asks to "sync my contacts", "sync contacts from my calendar", "add the people I met with to Notion", "log today's meetings as contacts", "who did I meet with this week and are they in my CRM", "catch up my contacts", or mentions getting calendar attendees into Notion. Works for today, a specific day, or a date range. Trigger this even if Sophia doesn't say "Notion" or "CRM" explicitly — any request to turn meetings/attendees into contact records belongs here.
---

Sync external meeting attendees from Sophia's calendar into her Notion Contacts database: create new contacts, enrich existing ones, and resolve duplicates. Unlike the unattended scheduled version of this job, this runs **with Sophia in the loop** — so when something is ambiguous, ask her rather than guessing or silently deferring.

## Setup

- User: Sophia Edwards (sophia@unlockinganalytics.com)
- Internal domain: `@unlockinganalytics.com`
- Timezone: America/Denver
- Notion Contacts data source ID: `2a9da677-edec-80f8-af29-000bfad2bc71` (use `collection://2a9da677-edec-80f8-af29-000bfad2bc71` for searches)
- Companies data source ID: `collection://2cdda677-edec-8067-a57f-000b64d32b9d`

## Step 1: Establish the date range

Default to **today** (America/Denver). Do NOT rely on the system-provided `currentDate` context variable — it reflects UTC and will be wrong for Mountain Time after 6 PM UTC. Instead, determine today's America/Denver date by calling the calendar API with a narrow window around the current moment and reading the timezone offset from the response, or by asking Sophia if there is any ambiguity.

If Sophia named a different window, honor it:
- "yesterday", "last Tuesday", a specific date → that single day
- "this week", "the last 3 days", "since Monday" → that range

If her phrasing is vague about timing ("recent meetings", "lately") and it materially changes the result, ask a one-line clarifying question before pulling the calendar. Otherwise just proceed and state the range you used.

Pull events from **both** calendars using `mcp__721da59c-2d48-44e6-97de-fd9e5a1d6ca7__list_events`, startTime = range start 00:00 America/Denver, endTime = range end 23:59 America/Denver:
- Primary business calendar (default, no `calendarId` needed)
- Personal calendar: `calendarId = sophiasull@gmail.com`

Deduplicate by event title + start time — the same meeting may appear on both calendars.

## Step 2: Filter events

Process only events with at least one attendee whose email is **not** `@unlockinganalytics.com` and **not** `sophiasull@gmail.com`. Skip:
- Solo blocks (no external attendees)
- Personal/recurring blocks whose summary matches "Organize thoughts", "Focus time", "Lunch", "Block", "Hold", "OOO"
- Cancelled events (status != "confirmed")

## Step 3: Look up each external attendee

For each external attendee:

**a. By email first.** Use `notion-search` with `data_source_url: collection://2a9da677-edec-80f8-af29-000bfad2bc71`, querying the attendee's email. Check both `Business Email` and `Personal Email` on returned pages.

**b. By name if no email match.** Try the calendar display name — both an in-data-source search and a workspace search.

**c. Watch for duplicate stubs.** Notion's meeting integration sometimes auto-creates contact pages with mis-transcribed names: a recently-created page in Contacts whose name is close-but-not-exact, with empty properties but meeting-note content. Treat these as ambiguous.

## Step 4: Classify and act

Sort each person into one bucket:

**MATCHED** — a single clear record. Enrich empty fields only; never overwrite. Update `Latest Contact Date` to the meeting date. If the meeting notes contain action items for Sophia, set `Follow-up Needed = __YES__` and append (don't replace) to `What to follow up on`. Read existing properties first to confirm a field is empty before writing.

**NEW** — no plausible match. Create a contact, gathering context from:
- The Notion meeting note for that meeting (`notion-query-meeting-notes` filtered by the meeting date, or `notion-search` for the title)
- Gmail intro/referral threads — `mcp__8fcdfe8a-629b-4634-8fd8-9baeacc12b12__search_threads` for the person's email, looking for intro/introduction subjects in the last ~30 days. Set `How/Where Met = "Referral"` if found.
- The calendar event description (booking-link descriptions sometimes carry phone/info)

**AMBIGUOUS** — a possible duplicate, partial match, or close-name stub. **This is where the in-the-loop part matters: don't just defer it.** Present the conflict to Sophia and let her decide — e.g. "I think this might be the existing record [Name + link], but the email doesn't match. Update that one, create a new contact, or skip?" Then act on her answer in the same run. Only fall back to flagging-without-resolution if she's stepped away or doesn't respond.

If Sophia is reviewing a long list, it's fine to batch the ambiguous cases and ask about them together rather than interrupting after each one.

## Contacts schema (data source `2a9da677-edec-80f8-af29-000bfad2bc71`)

- `Name` (title) — required
- `Business Email`, `Personal Email` (email)
- `Phone` (phone_number), `LinkedIn`, `Website`, `Address`, `Booking Calendar` (url/text)
- `Company` (relation to `collection://2cdda677-edec-8067-a57f-000b64d32b9d`) — only set on an unambiguous Companies match
- Date properties use expanded form: `date:First Contact Date:start` (ISO date), `date:First Contact Date:is_datetime` = 0. Same for `Latest Contact Date`.
- `Follow-up Needed`, `Inactive` (checkbox): `__YES__` or `__NO__`
- `How/Where Met` (select): one of `Dames`, `LI reachout`, `CDO NYC Nov 2025`, `Referral`, `Cold outreach`, `While at IC`, `INFORMS`, `Teradata`, `Rose-Hulman`, `SheLEADS`, `Northwestern`, `Networking Meetup`, `Chamber of Commerce`. Leave blank if not confidently inferable.
- `Type` (select): one of `Recruiter`, `Peer`, `Manager`, `Client`, `Mentor`, `Direct Report`, `Vendor`, `Partner`, `Investor`, `Pro Bono`, `Estate Planner`, `Mortgage`, `Financial Planner`, `Tax Accountant`, `Marketing`. Leave blank if not confidently inferable. **Rule:** anyone whose `Company` resolves to an existing client in the Companies database should be set to `Client`, regardless of their individual role.
- `What to follow up on` (text)

For new contacts: `notion-create-pages` with `parent: {type: "data_source_id", data_source_id: "2a9da677-edec-80f8-af29-000bfad2bc71"}`. Page body should include a brief Background section, an Action Items list, and key context from the meeting notes.

For updates: `notion-update-page` with `command: "update_properties"`.

## Output

Report concisely:

```
## Contact Sync — [date or range]

**Events processed:** N

**Created (M):**
- [Name] ([email]) — [Notion link] — source: [meeting / referral from X]

**Enriched (K):**
- [Name] — [link] — updated: [fields touched]

**Resolved with you (J):**
- [Name] — [what you decided]

**Skipped events (S):** [titles, one per line]
```

If nothing qualifies: "No external meetings in that range — nothing to sync."

## Hard rules

- Never overwrite a non-empty field.
- Never delete a page or property.
- Never create a record when there's a plausible existing match unresolved — ask Sophia instead.
- If a tool call fails, retry once with a simplified query, then surface the error rather than retrying indefinitely.
