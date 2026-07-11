# IA3 - Use-Case-to-Story Traceability Card

## Student details

- Student name: Asare Adom Kwaku
- Student ID: 22304024
- GitHub username: Adom-Asare
- Team name: SThrive
- Project title: SThrive Network — She Thrives Network Platform
- Client/project domain: Faith-led women's professional networking and membership platform (UK-registered charity)
- Date submitted: 13 July 2026

## 1. Selected use case

- Use case name: UC12 — RSVP to Free Event
- Source evidence: D2 SRS (3 July 2026) Section 2 UC12, Section 3 US14, Section 4 NFR-P02; D3 Sprint Backlog story US14-A (5 points, Must, Sprint 3); client meeting transcript 26 June 2026 with Dr. Kinsella — confirmed events are admin-created and open to all members, RSVP and attendance tracking identified as a core requirement replacing the current WhatsApp-based manual process; team GitHub issue #35: https://github.com/Dept-of-Comp-Sci-University-of-Ghana/dcit208-client-project-2026-team-engineering-repository-sthrive/issues/35
- Why this use case matters to the client: She Thrives Network currently announces events across WhatsApp, Instagram, and word of mouth with no formal way to track who is attending or manage capacity. Members express interest informally and leadership has no visibility into actual attendance numbers. A reliable RSVP system directly replaces this manual, unstructured process and gives Dr. Kinsella's team real data on which events attract engagement. The atomic capacity check is critical because event announcements go out to the full WhatsApp community simultaneously — multiple members attempting to RSVP at the same moment for a limited-capacity session is a realistic, not hypothetical, load scenario for this platform.

## 2. Use case card

- Actor: Free Member / Pro/Elite Member (primary) | Supabase/PostgreSQL, Email Service — Resend/SendGrid (secondary)
- Goal: Reserve a spot at a free event and receive confirmation, or be placed on a waitlist if the event is at capacity
- Trigger: Authenticated member clicks the RSVP button on an event detail page
- Preconditions: Member is authenticated with a valid JWT. Event exists in the database with status: scheduled. Member has not already RSVPd to this event. Event ticket price is null or zero (free event).

### Main success flow

1. Member opens the event calendar, selects an event, and views the event detail page.
2. Member clicks the RSVP button. The Next.js frontend sends POST /api/events/[id]/rsvp with the member's JWT in the Authorization header.
3. Auth middleware verifies the JWT and returns 401 if invalid or expired. Upstash Redis rate-limiting middleware checks request frequency per IP.
4. The API route executes an atomic SQL UPDATE on the events table: increment confirmed_count only if confirmed_count < capacity. Because both the check and the increment happen in a single indivisible database statement, no two concurrent requests can both pass the capacity check simultaneously.
5. The UPDATE returns the updated row — a capacity slot was available. The API inserts an RSVP record with status: confirmed, returns 201 with { success: true, status: "confirmed", rsvpId }, and queues a confirmation email asynchronously.
6. The frontend displays "You are registered for this event!" and replaces the RSVP button with a confirmed status label.

### Alternate flow

1. The atomic UPDATE returns nothing — confirmed_count has already reached capacity. The API inserts an RSVP record with status: waitlisted and returns 200 with { success: true, status: "waitlisted", rsvpId }.
2. The frontend displays "You have been added to the waitlist." If a confirmed attendee later cancels, the system automatically promotes the first waitlisted member (ordered by createdAt) to confirmed and sends them a notification email.

### Exception / failure flow

1. Member has already RSVPd: the compound unique index on (event_id, user_id) catches this at the database level regardless of application-level checks. The API returns 409 "You have already registered for this event." No duplicate RSVP record is created.
2. Supabase connection drops mid-transaction: PostgreSQL rolls back the transaction automatically — confirmed_count is never left in a partially incremented state. The API returns 503 "Service temporarily unavailable, please try again" rather than hanging. The RSVP button returns to its default state so the member can retry.

### Postconditions

- An RSVP record exists in the rsvps table with status: confirmed or status: waitlisted.
- The event's confirmed_count has been incremented by exactly one on the confirmed path only.
- A confirmation or waitlist email has been sent or queued for the member.

## 3. User story

As a **member**, I want to **RSVP to a free event with a reliable capacity check**, so that **my spot is properly reserved even under high demand**.

## 4. Acceptance criteria

### AC1 - Successful path
Given a member is authenticated and the event has available capacity, when they click RSVP and the request is processed, then the system creates an RSVP record with status: confirmed, increments the event's confirmed_count by exactly one, returns a 201 response, and queues a confirmation email to the member.

### AC2 - Error or exception path
Given two members submit RSVP requests simultaneously for the last available spot, when both requests reach the database at the same moment, then the atomic database operation ensures exactly one RSVP is created with status: confirmed and exactly one is created with status: waitlisted — the event's confirmed_count equals capacity, not capacity + 1, and no overbooking occurs.

### AC3 - Duplicate prevention
Given a member who has already RSVPd attempts to RSVP again, when the request is made, then the system returns 409 with the message "You have already registered for this event" and no duplicate record is created.

### AC4 - Waitlist promotion
Given a confirmed attendee cancels their RSVP, when the cancellation is processed, then the first waitlisted member by createdAt is automatically promoted to confirmed and receives a notification email.

## 5. Non-functional requirement link

- NFR category: Performance — Correctness Under Concurrency
- NFR statement (NFR-P02): The RSVP capacity check must be implemented as a single atomic database operation. The naive check-then-write pattern — read confirmed_count, check against capacity, then increment in separate steps — is explicitly prohibited, as it produces a race condition under concurrent load. The implementation must use Supabase's PostgreSQL atomic UPDATE with a WHERE condition to guarantee that the capacity threshold is enforced at the database level, not only the application level, regardless of how many server instances are running simultaneously.
- Why it matters for this use case: She Thrives Network announces events to the full WhatsApp community simultaneously. When a popular session goes live, multiple members will click RSVP within the same second. Without an atomic implementation, two requests can both read confirmed_count = 49 (capacity = 50), both pass the check, and both write — producing 51 confirmed attendees for a 50-person event. This is not a theoretical concern: it is a realistic load pattern for this community and a direct consequence of how the organisation currently communicates. The atomic SQL UPDATE eliminates the gap between check and write at the database level, making overbooking structurally impossible.

## 6. GitHub issue link

- Team repository issue title: [Story] US14-A: RSVP atomic capacity check #35
- Team repository issue URL: https://github.com/Dept-of-Comp-Sci-University-of-Ghana/dcit208-client-project-2026-team-engineering-repository-sthrive/issues/35

## 7. Test idea

- Test name: Concurrent RSVP Race Condition — Last Available Slot
- Input/context: A test event exists in the database with capacity: 1 and confirmed_count: 0. Two distinct authenticated test users (User A and User B) both send POST /api/events/[id]/rsvp simultaneously using Promise.all() to fire both requests at the same millisecond.
- Expected result: Exactly one RSVP record has status: confirmed. Exactly one RSVP record has status: waitlisted. The event's confirmed_count equals 1, not 2. No duplicate confirmed RSVPs exist in the rsvps table for this event. If the atomic implementation is correct, this test passes 100% of the time regardless of request timing — intermittent failure would indicate the race condition is still present.

## 8. Request-flow thinking

- Device/interface: Any modern browser (desktop or mobile). SThrive Network is a responsive web application, not a native mobile app. Member is viewing the event detail page in the Next.js frontend.
- Route/screen/endpoint: Frontend screen: /events/[id]. Backend API route: POST /api/events/[id]/rsvp.
- Data sent: HTTPS POST request. Authorization header: Bearer [JWT access token]. URL path parameter: event ID. No request body required for free events — the member's identity is carried in the JWT, not a body field.
- Validation point: Three layers. (1) Auth middleware verifies the JWT and rejects with 401 if invalid or expired. (2) Upstash Redis rate-limiting checks request frequency per IP. (3) The atomic SQL UPDATE enforces capacity at the database level — this is the critical validation that prevents overbooking regardless of application-level logic.
- Database table, collection or external service touched: Supabase PostgreSQL — events table (atomic UPDATE on confirmed_count); rsvps table (INSERT with confirmed or waitlisted status). Unique compound index on (event_id, user_id) prevents duplicate RSVPs. Email service (Resend/SendGrid) triggered asynchronously for confirmation notification.
- Response returned: Confirmed: 201 Created { success: true, status: "confirmed", rsvpId }. Waitlisted: 200 OK { success: true, status: "waitlisted", rsvpId }. Duplicate RSVP: 409 Conflict. Database failure: 503 Service Unavailable — server fails fast and clearly, never hangs.
- Failure point: The most likely failure point is the Supabase connection dropping mid-transaction. PostgreSQL's transaction guarantees that a dropped connection rolls back automatically — confirmed_count is never left in a partially incremented state. The API catches the connection error and returns 503 rather than leaving the request hanging until the browser times out.
- Security/privacy concern: The RSVP endpoint is not publicly accessible — JWT verification runs before any database operation. Unauthenticated or tampered requests are rejected at the middleware layer with 401 before touching the database. The event ID in the URL is validated against the database to prevent RSVPs being submitted for non-existent, cancelled, or completed events. No payment data is handled at this endpoint — ticketed events redirect to Stripe's hosted checkout, so no card data ever passes through the platform's own API.

## 9. AI disclosure

- AI used? Yes
- Tool used, if any: Claude (Anthropic)
- What AI helped with: Structuring the traceability card to match the IA3 template format; verifying technical accuracy of the atomic SQL operation and the 9-step request-flow explanation; cross-checking that all required parts were addressed per the assignment brief; suggesting additional acceptance criteria beyond the initial AC1 and AC2.
- What I changed: All content was verified against the team's own deliverables — D2 SRS (3 July 2026), D3 Sprint Backlog, and the client meeting transcript of 26 June 2026 with Dr. Kinsella. The user story was taken verbatim from the actual GitHub issue #35, not from AI-generated text. The failure point description (Supabase connection rollback behaviour) was grounded in real debugging experience from the ThinkBoard project. Status codes (201, 200, 409, 503) were verified against the team's architecture document.
- What I personally verified: UC12 traces directly to US14-A in the D2 SRS and to GitHub issue #35 in the team repository. NFR-P02 is taken verbatim from Section 4 of the team's D2 SRS. The atomic SQL UPDATE pattern was discussed and understood in full during team architecture planning sessions. I can explain the race condition, the atomic fix, the waitlist promotion flow, and the failure behaviour orally.

## 10. Integrity declaration

I understand the use case, user story, acceptance criteria, traceability links and request-flow reasoning submitted here, and I can explain them orally if called upon.

Typed name: Asare Adom Kwaku