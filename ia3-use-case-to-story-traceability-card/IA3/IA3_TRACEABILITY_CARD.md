# IA3 - Use-Case-to-Story Traceability Card

## Student details

- Student name: Asare Adom Kwaku
- Student ID: 22304024
- GitHub username: Adom-Asare
- Team name: SThrive
- Project title: SThrive Network — She Thrives Network Platform
- Client/project domain: Faith-led women's professional networking and membership platform (UK charity)
- Date submitted: July 2026

## 1. Selected use case

- Use case name: UC12 — RSVP to Free Event
- Source evidence: D2 SRS (Section 2, UC12), D3 backlog (US14-A, US14-B), client meeting transcript 26 June 2026 (Dr. Kinsella confirmed events are admin-created and open to members; RSVP functionality confirmed as core MVP feature), team GitHub issue: [paste US14 issue URL]
- Why this use case matters to the client: She Thrives Network currently announces events manually across WhatsApp, Instagram, and word of mouth with no way to track who is attending or manage capacity. A working RSVP system with automatic waitlist promotion directly replaces that manual process and gives leadership real visibility into event attendance — one of the core pain points confirmed in the client meeting.

## 2. Use case card

- Actor: Free Member / Pro/Elite Member (primary); Supabase/PostgreSQL (secondary — atomic write); Email Service / Resend (secondary — confirmation notification)
- Goal: Reserve a spot at a free event and receive confirmation, or be placed on a waitlist if the event is at capacity
- Trigger: Member clicks the RSVP button on an event detail page
- Preconditions: Member is authenticated with a valid JWT. Event exists in the database with status: scheduled. Member has not already RSVPd to this event. Event is free (ticketPrice is null or zero).

### Main success flow

1. Member views the event calendar, opens an event, and clicks "RSVP."
2. The Next.js frontend sends POST /api/events/[id]/rsvp with the member's JWT in the Authorization header.
3. Auth middleware verifies the JWT; rate-limiting middleware checks request frequency via Upstash Redis.
4. The API route executes an atomic SQL UPDATE on the events table: increment confirmed_count only if confirmed_count < capacity. Because both the check and the increment happen in a single indivisible database statement, no two concurrent requests can both pass the capacity check simultaneously.
5. The UPDATE returns the updated row (capacity slot was available): the API inserts an RSVP record with status: confirmed, returns 201 with `{ success: true, status: "confirmed", rsvpId }`, and triggers a confirmation email asynchronously.
6. The frontend displays "You are registered for this event!" and replaces the RSVP button with a confirmed status label.

### Alternate flow

1. The atomic UPDATE returns nothing (confirmed_count has already reached capacity): the API inserts an RSVP record with status: waitlisted and returns 200 with `{ success: true, status: "waitlisted", rsvpId }`.
2. The frontend displays "You have been added to the waitlist." If a confirmed attendee later cancels, the system automatically promotes the first waitlisted member (ordered by createdAt) to confirmed and sends them a notification email.

### Exception / failure flow

1. Member has already RSVPd: the compound unique index on (event_id, user_id) catches this at the database level regardless of application-level checks; API returns 409 with "You have already registered for this event." No duplicate RSVP record is created.
2. Supabase connection drops mid-transaction: PostgreSQL rolls back the transaction automatically — the confirmed_count is never partially incremented. The API returns 503 "Service temporarily unavailable, please try again" rather than hanging. The RSVP button returns to its default state so the member can retry.

### Postconditions

- An RSVP record exists in the rsvps table with status: confirmed or status: waitlisted.
- The event's confirmed_count has been incremented by exactly one (confirmed path only).
- A confirmation or waitlist email has been sent or queued for the member.

## 3. User story

As a **member**, I want to **RSVP to a free event**, so that **my spot is reserved and I receive confirmation of my attendance.**

## 4. Acceptance criteria

### AC1 - Successful path
Given a member is authenticated and the event has available capacity, when they submit an RSVP request, then the system creates an RSVP record with status: confirmed, increments the event's confirmed_count by exactly one, returns a 201 response, and sends a confirmation email.

### AC2 - Error or exception path (concurrent race condition)
Given two members submit RSVP requests simultaneously for the last available spot, when both requests reach the database at the same moment, then the atomic database operation ensures exactly one RSVP is created with status: confirmed and exactly one is created with status: waitlisted — the event's confirmed_count equals capacity, not capacity + 1, and no overbooking occurs.

### AC3 - Already RSVPd
Given a member who has already RSVPd attempts to RSVP again, when the request is made, then the system returns 409 with the message "You have already registered for this event" and no duplicate record is created.

### AC4 - Waitlist promotion
Given a confirmed attendee cancels their RSVP, when the cancellation is processed, then the first waitlisted member (by createdAt) is automatically promoted to confirmed and receives a notification email.

### AC5 - Email service failure
Given the email service is unavailable at the time of RSVP confirmation, when the RSVP is saved, then the RSVP record is still created successfully and the email is queued for retry — the member is not shown an error and the RSVP is not rolled back.

## 5. Non-functional requirement link

- NFR category: Performance — Correctness Under Concurrency
- NFR statement (NFR-P02): The RSVP capacity check must be implemented as a single atomic database operation. The naive "check-then-write" pattern is explicitly prohibited. The implementation must use Supabase's PostgreSQL atomic UPDATE with a WHERE condition to guarantee that the capacity threshold is enforced at the database level, not only the application level, regardless of how many server instances are running simultaneously.
- Why it matters for this use case: She Thrives Network runs events for a professional community where capacity is a real constraint. If the RSVP logic uses a non-atomic "read confirmedCount, check against capacity, then write" pattern, two members clicking RSVP at the same instant can both pass the check before either has written — resulting in overbooking. Under the load of a popular event announcement (which the client currently spreads across WhatsApp, Instagram, and church), this race condition is not hypothetical. The atomic SQL statement is the only correct fix because it eliminates the gap between check and write at the database level.

## 6. GitHub issue link

- Team repository issue title: [US14] RSVP to Free Event — atomic capacity check + waitlist promotion
- Team repository issue URL: [paste actual GitHub issue URL from the SThrive team board]

## 7. Test idea

- Test name: Concurrent RSVP Race Condition — Last Available Slot
- Input/context: A test event exists in the database with capacity: 1 and confirmed_count: 0. Two distinct authenticated test users (User A and User B) both send POST /api/events/[id]/rsvp simultaneously using Promise.all() to fire both requests at the same millisecond.
- Expected result: Exactly one RSVP record has status: confirmed. Exactly one RSVP record has status: waitlisted. The event's confirmed_count equals 1, not 2. No duplicate confirmed RSVPs exist in the rsvps table for this event. If the atomic implementation is correct, this test passes 100% of the time regardless of request timing — intermittent failure would indicate the race condition is still present.

## 8. Request-flow thinking

- Device/interface: Any modern browser (desktop or mobile) — SThrive Network is a responsive web application, not a native mobile app. Member is viewing the event calendar page in the React/Next.js frontend.
- Route/screen/endpoint: Frontend screen: /events/[id]. Backend endpoint: POST /api/events/[id]/rsvp. The member's identity comes from the JWT in the Authorization header — no sensitive body data is required for a free event RSVP.
- Data sent: HTTPS POST request. Headers: Authorization: Bearer [JWT access token]. URL param: event ID. No request body required for free events (ticketed events would include a payment reference).
- Validation point: Two layers. (1) Auth middleware verifies the JWT and rejects with 401 if invalid or expired. (2) Upstash Redis rate limiting checks request frequency per IP. (3) The atomic SQL UPDATE itself enforces capacity at the database level — this is the critical validation that prevents overbooking regardless of application-level logic.
- Database table, collection or external service touched: Supabase PostgreSQL — events table (atomic UPDATE on confirmed_count), rsvps table (INSERT with confirmed or waitlisted status). Unique compound index on (event_id, user_id) prevents duplicate RSVPs. Email service (Resend/SendGrid) triggered asynchronously for confirmation notification.
- Response returned: Confirmed path: 201 Created with `{ success: true, status: "confirmed", rsvpId }`. Waitlisted path: 200 OK with `{ success: true, status: "waitlisted", rsvpId }`. Duplicate RSVP: 409 Conflict. Database failure: 503 Service Unavailable — server fails fast and clearly, never hangs.
- Failure point: The most likely failure point is the Supabase connection dropping mid-transaction. PostgreSQL's transaction guarantees that a dropped connection rolls back automatically — the confirmed_count is never left in a partially incremented state. The API catches the connection error and returns 503 rather than leaving the request hanging until the browser times out.
- Security/privacy concern: The RSVP endpoint is not publicly accessible — JWT verification runs before any database operation. An unauthenticated or tampered request is rejected at the middleware layer with 401 before touching the database. The event ID in the URL is validated against the database to prevent RSVPs being submitted for non-existent, cancelled, or completed events. No payment data is handled at this endpoint — ticketed events redirect to Stripe's hosted checkout, so no card data ever passes through the platform's own API.

## 9. AI disclosure

- AI used? Yes
- Tool used, if any: Claude (Anthropic)
- What AI helped with: Structuring the traceability card format, verifying the technical accuracy of the atomic SQL operation and the request-flow step sequence, ensuring the content aligns with the team's D2 SRS and D3 backlog, and cross-checking that all 12 required fields were covered.
- What I changed: All content was reviewed against the team's own deliverables (D2 SRS, D3 backlog, client meeting transcript of 26 June 2026 with Dr. Kinsella). Specific technical details (the SQL UPDATE pattern, the 503 vs 409 status codes, the Supabase transaction rollback behaviour on connection failure) were verified against the team's architecture document and my own understanding of the system. Phrasing was adjusted to reflect the actual stack (Next.js + Supabase, not Express + MongoDB) and the confirmed client context.
- What I personally verified: The use case and user story trace directly to UC12 and US14 in the D2 SRS. The NFR-P02 statement is taken verbatim from the team's NFR section. The failure point (Supabase connection dropping mid-transaction) is grounded in real experience with database connection failures encountered during the ThinkBoard project earlier in the year. The concurrent test idea reflects the actual race condition we identified and resolved in the architecture document.

## 10. Integrity declaration

I understand the use case, user story, acceptance criteria, traceability links and request-flow reasoning submitted here, and I can explain them orally if called upon.

Typed name: Asare Adom Kwaku