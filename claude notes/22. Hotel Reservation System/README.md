# System Design: Hotel Reservation System (Beginner-Friendly Guide)

---

## What Are We Building?

Think of Marriott.com or Booking.com — a hotel chain's website where you can:
- Browse hotels and see room details
- Check which room types are available for your dates
- Reserve a room and pay
- Cancel a reservation

The interesting engineering challenges hidden in this simple flow:
- **Double booking** — what if two people book the last available room at the exact same moment?
- **Overbooking** — hotels intentionally sell more rooms than exist (expecting cancellations)
- **Payment consistency** — what if the server crashes after charging a card but before confirming the booking?
- **Scale** — how do we handle rush bookings (concert/event near the hotel)?

---

## Step 1: Design Scope

**Scale:**
| Parameter | Value |
|-----------|-------|
| Hotels | 5,000 |
| Total rooms | 1,000,000 |
| Occupied rooms (avg) | 70% |
| Average stay | 3 days |
| Daily reservations | 1M × 0.7 / 3 ≈ **240,000/day** |
| Reservation QPS | 240,000 / 86,400 ≈ **3/sec** (very low!) |

**Key feature:** Hotel prices change daily. Same room type may cost $150 on Friday and $89 on Sunday.

**QPS Funnel:**

<img src="../../system-design-notes/22. Hotel Reservation System/images/qps-estimation.png" alt="QPS Estimation" width="450">

```
View hotel/room page:  QPS = 300  (browsing)
Order booking page:    QPS = 30   (considering)
Reserve room:          QPS = 3    (actually books)
```

Most traffic is read-heavy browsing. Actual reservations are rare.

**Non-functional requirements:**
- Handle concurrency (multiple people booking same room)
- Support overbooking (hotel sells 110% of rooms to account for cancellations)
- Moderate latency — a few seconds to process a reservation is acceptable

---

## Step 2: API Design

**Hotel & Room APIs (public read, admin write):**
```
GET    /v1/hotels/{id}              ← hotel details
POST   /v1/hotels                   ← add hotel (admin only)
GET    /v1/hotels/{id}/rooms/{id}   ← room details
POST   /v1/hotels/{id}/rooms        ← add room (admin only)
```

**Reservation APIs:**
```
GET    /v1/reservations             ← current user's reservation history
GET    /v1/reservations/{id}        ← specific reservation details
POST   /v1/reservations             ← make a new reservation
DELETE /v1/reservations/{id}        ← cancel a reservation
```

**Make reservation request body:**
```json
{
  "startDate":    "2026-06-01",
  "endDate":      "2026-06-04",
  "hotelID":      "245",
  "roomTypeID":   "12354673389",
  "roomCount":    2,
  "reservationID":"13422445"
}
```

> **Note:** `reservationID` is an **idempotency key** — more on this in the concurrency section. And we reserve a **room type** (e.g., "King Suite"), not a specific room number. The room number is assigned at check-in.

---

## Step 3: Database Choice — Why Relational?

| Requirement | Why SQL wins |
|-------------|-------------|
| ACID transactions | Critical for preventing double-booking; we need atomic check-then-update |
| Read-heavy, low-write | SQL is optimized for reads; reservations QPS is only 3/sec |
| Clear data relationships | Hotels → rooms → reservations are naturally relational |
| Complex queries | "Find available room types for dates X–Y" is a date-range join — easy in SQL |

> **Why not NoSQL?** Cassandra/DynamoDB excel at high-write workloads. Our write rate (3 reservations/sec) is tiny. We'd lose ACID guarantees that prevent double-booking, for no performance benefit.

---

## Step 4: Data Schema

### Initial Schema (v1)

<img src="../../system-design-notes/22. Hotel Reservation System/images/schema-design.png" alt="Schema Design" width="500">

### Reservation Status State Machine

<img src="../../system-design-notes/22. Hotel Reservation System/images/status-state-machine.png" alt="Status State Machine" width="400">

```
Pending → Paid → Refunded
       → Cancelled
       → Rejected
```

### Problem with v1

In v1, users reserve a specific `room_id`. But in hotels, you don't pick room 412 — you pick "King Suite" and a room is assigned at check-in. We need to track **room type inventory**, not individual rooms.

### Updated Schema (v2)

<img src="../../system-design-notes/22. Hotel Reservation System/images/updated-schema.png" alt="Updated Schema" width="500">

The key new table is **`room_type_inventory`**:

| hotel_id | room_type_id | date       | total_inventory | total_reserved |
|----------|--------------|------------|-----------------|----------------|
| 211      | 1001         | 2026-06-01 | 100             | 80             |
| 211      | 1001         | 2026-06-02 | 100             | 82             |
| 211      | 1001         | 2026-06-03 | 100             | 86             |
| 211      | 1002         | 2026-06-01 | 200             | 16             |

**One row per (hotel, room type, date).** Pre-populated by a daily CRON job.

**Availability check:**
```sql
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ?  AND hotel_id = ?
AND date BETWEEN '2026-06-01' AND '2026-06-03'
```

**Reservation is allowed when** (with 10% overbooking support):
```
(total_reserved + numberOfRoomsToBook) <= 110% × total_inventory
```

**Storage estimate:**
```
5,000 hotels × 20 room types × 2 years × 365 days = 73 million rows
```
73 million rows fits comfortably on a single MySQL server. Read replicas add availability.

---

## Step 5: High-Level Architecture

<img src="../../system-design-notes/22. Hotel Reservation System/images/high-level-design.png" alt="High Level Design" width="500">

**Microservice architecture:**

```
User (phone/browser)
        ↓
      [CDN]             ← static assets (JS, images)
        ↓
[Public API Gateway]    ← auth, rate limiting
  ↓           ↓           ↓           ↓
[Hotel     [Rate      [Reservation  [Payment
 Service]   Service]   Service]      Service]
  ↓ + Cache  ↓           ↓             ↓
[Hotel DB] [Rate DB] [Reservation DB] [Payment DB]

Admin →  [Internal API] → [Hotel Management Service]
```

| Service | Responsibility |
|---------|---------------|
| **Hotel Service** | Hotel and room static data (names, photos, amenities) — cached aggressively |
| **Rate Service** | Room prices per date (changes daily based on occupancy) |
| **Reservation Service** | Handles bookings, tracks inventory (`room_type_inventory` table) |
| **Payment Service** | Charges cards, handles refunds, updates reservation status |
| **Hotel Management Service** | Admin functions: view/cancel reservations, update hotel info |

### Microservices vs. Monolith

<img src="../../system-design-notes/22. Hotel Reservation System/images/microservices-vs-monolith.png" alt="Microservices vs Monolith" width="500">

> **Our hybrid approach:** Reservation and inventory are handled by the **same service** sharing the **same database**. This lets us use a single SQL transaction for "check availability + reserve room" — crucial for preventing double-booking. Pure microservices with separate DBs would require distributed transactions (complex and slow).

---

## Step 6: Concurrency Problem — Double Booking

This is the core engineering challenge. Two scenarios:

### Scenario 1: Same User Clicks "Book" Twice

<img src="../../system-design-notes/22. Hotel Reservation System/images/double-booking-single-user.png" alt="Double Booking Single User" width="450">

User clicks "Book" → browser freezes → clicks again → two reservation requests sent.

**Solutions:**
1. **Client-side:** Disable the button after first click (but JavaScript can be disabled)
2. **Idempotency key** (the real fix):

<img src="../../system-design-notes/22. Hotel Reservation System/images/idempotency.png" alt="Idempotency" width="450">

**How it works:**
1. When user opens the booking page, server generates a unique `reservation_id` (UUID) and returns it
2. User fills in details and clicks "Book" — request includes this `reservation_id`
3. If user clicks again, same `reservation_id` is sent
4. The `reservation_id` column has a **UNIQUE constraint** in the database

<img src="../../system-design-notes/22. Hotel Reservation System/images/unique-constraint-violation.png" alt="Unique Constraint Violation" width="450">

Second insert with the same `reservation_id` → database rejects it with a unique constraint violation → only one reservation created.

> **Why generate the ID server-side on page open?** If the client generates it, a malicious user could manipulate it. Server-generated IDs are trusted. Also, generating it upfront means the ID exists before the user submits, preventing true duplicates.

### Scenario 2: Two Different Users Book the Last Room

<img src="../../system-design-notes/22. Hotel Reservation System/images/double-booking-multiple-users.png" alt="Double Booking Multiple Users" width="450">

```
Hotel has: total_inventory=100, total_reserved=99 (1 room left)

User 1: reads total_reserved=99  ← "1 room available!"
User 2: reads total_reserved=99  ← "1 room available!"
User 1: sets total_reserved=100  ← commits ✅
User 2: sets total_reserved=100  ← commits ✅  ← DOUBLE BOOKING!
```

Both users see 1 room available. Both book it. Both succeed. **Two reservations for one room.**

---

## Step 7: Solving Double Booking — Three Options

### Option 1: Pessimistic Locking

<img src="../../system-design-notes/22. Hotel Reservation System/images/pessimistic-locking.png" alt="Pessimistic Locking" width="500">

Use `SELECT ... FOR UPDATE` to lock the row while a transaction is in progress:

```sql
BEGIN TRANSACTION;
  -- Lock the row — no one else can read or write it until we commit
  SELECT total_inventory, total_reserved
  FROM room_type_inventory
  WHERE hotel_id = 211 AND room_type_id = 1001 AND date = '2026-06-01'
  FOR UPDATE;

  -- Check if room is available
  -- If yes, reserve it
  UPDATE room_type_inventory
  SET total_reserved = total_reserved + 1
  WHERE hotel_id = 211 AND room_type_id = 1001 AND date = '2026-06-01';
COMMIT;
```

Transaction 2 **waits** while Transaction 1 holds the lock. After Transaction 1 commits, Transaction 2 runs — but now sees `total_reserved=100` = no rooms left → rollback.

**Pros:** Simple; guarantees no double-booking  
**Cons:** Serializes all bookings for that room type → poor throughput; risk of deadlocks if multiple rows locked; bad for long-running transactions

> **Not recommended** for hotel bookings due to scalability issues.

### Option 2: Optimistic Locking ✅ Recommended

<img src="../../system-design-notes/22. Hotel Reservation System/images/optimistic-locking.png" alt="Optimistic Locking" width="500">

No lock held. Instead, add a `version` column. When updating, check that the version hasn't changed:

```sql
-- Read current state
SELECT total_reserved, version FROM room_type_inventory
WHERE hotel_id = 211 AND room_type_id = 1001 AND date = '2026-06-01';
-- Returns: total_reserved=99, version=5

-- Try to update — only succeeds if version is still 5
UPDATE room_type_inventory
SET total_reserved = 100, version = 6
WHERE hotel_id = 211 AND room_type_id = 1001
  AND date = '2026-06-01'
  AND version = 5;  -- ← version check!

-- If 0 rows affected: someone else updated first → retry
```

**Scenario:**
- User 1 reads version=5, updates to version=6 ✅
- User 2 also read version=5, tries to update with `AND version=5` → 0 rows affected ❌ → retry → reads new state (total_reserved=100) → "no rooms available"

**Pros:** No database locks; high read throughput; no deadlocks  
**Cons:** High contention (many retries) degrades performance; not suitable for very hot rooms

> **Good choice for hotel reservations** — most room types have low-to-moderate contention most of the time.

### Option 3: Database Constraints

<img src="../../system-design-notes/22. Hotel Reservation System/images/database-constraint.png" alt="Database Constraint" width="450">

Add a CHECK constraint to the database:

```sql
CONSTRAINT check_room_count
CHECK ((total_inventory - total_reserved) >= 0)
```

Now if both users try to update `total_reserved` to 100 (when `total_inventory=100`), the database enforces the constraint:
- User 1 updates: `100 - 100 = 0` ≥ 0 ✅
- User 2 updates: `100 - 101 = -1` < 0 ❌ → constraint violation → automatic rollback

**Pros:** Extremely simple to implement; guaranteed at the database level  
**Cons:** Harder to version-control than application code; not all databases support CHECK constraints; performs poorly under very high contention

> **Also a good choice** for hotel reservations given its simplicity and our relatively low QPS.

---

## Step 8: Scalability

The base system handles 5,000 hotels comfortably. But what if we grow to Booking.com scale?

### Database Sharding

<img src="../../system-design-notes/22. Hotel Reservation System/images/database-sharding.png" alt="Database Sharding" width="500">

Shard by `hotel_id` — all data for one hotel stays on one shard:

```
Shard key: hotel_id % 16
hotel_id = 245 → 245 % 16 = 5 → Shard 5
hotel_id = 211 → 211 % 16 = 3 → Shard 3
```

**Why `hotel_id`?** Every query filters by `hotel_id`. By sharding on it, queries never need to touch multiple shards.

**At large scale:** If QPS grows to 30,000, with 16 shards: 30,000 / 16 = ~1,875 QPS per shard — well within a MySQL cluster's capacity.

> **Exception:** A very popular hotel (Las Vegas resort with 5,000 rooms) could overload a single shard. Solution: dedicate a separate shard to mega-hotels.

### Inventory Cache (Redis)

<img src="../../system-design-notes/22. Hotel Reservation System/images/inventory-cache.png" alt="Inventory Cache" width="450">

```
Cache key:   hotelID_roomTypeID_date
Cache value: number of available rooms

Example: "211_1001_2026-06-01" → 20
```

**Flow:**
- Reservation Service **queries inventory**: check Redis cache first → fast
- Reservation Service **updates inventory**: write to database (source of truth) → database change event (via Debezium/CDC) → async update Redis cache

> **Why async cache update instead of write-through?** The database is the authoritative source for availability. The cache serves the high-read load (browsing). Eventual consistency in the cache is fine — the final booking transaction always validates against the database. A user might briefly see "2 rooms left" when it's actually 0, but the booking will correctly fail.

**Benefits of Redis cache:**
- Hotel browsing (QPS=300) served entirely from memory — database sees minimal read load
- Sub-millisecond availability lookups for room search
- TTL on past dates — old data automatically expires

---

## Step 9: Data Consistency Across Microservices

### The Problem with Pure Microservices

<img src="../../system-design-notes/22. Hotel Reservation System/images/microservices-vs-monolith.png" alt="Microservices vs Monolith" width="500">

In a pure microservice architecture, Inventory Service and Reservation Service have separate databases. "Reserve a room" must:
1. Check and update inventory (Inventory DB)
2. Create the reservation record (Reservation DB)
3. Charge the customer (Payment DB)

If step 1 succeeds but step 2 crashes → inventory is decremented but no reservation exists. Inconsistent state.

### Monolith: Single Transaction

<img src="../../system-design-notes/22. Hotel Reservation System/images/atomicity-monolith.png" alt="Atomicity Monolith" width="450">

In a monolith with one database, both operations are in one SQL transaction — either both succeed or both fail. Simple and safe.

### Microservice: Non-Atomic

<img src="../../system-design-notes/22. Hotel Reservation System/images/microservice-non-atomic-operation.png" alt="Microservice Non-Atomic Operation" width="450">

With separate services, there's no "one transaction" that spans both. Crash between steps → partial state.

### Our Pragmatic Solution

**Reservation + Inventory use the SAME service and the SAME database.** They share a single SQL transaction.

Only Payment is separate (it calls an external payment gateway anyway). The Reservation Service handles payment status asynchronously.

> **Purity vs. practicality:** A pure microservice purist would say "each service must own its database." But forcing distributed transactions for a 3 QPS reservation system adds enormous complexity for minimal benefit. Know when pragmatism wins.

### If You Must Go Full Microservices

Two patterns for cross-service consistency:

| Pattern | How | Trade-off |
|---------|-----|-----------|
| **Two-Phase Commit (2PC)** | Coordinator locks all services atomically | Strong consistency; slow; single coordinator failure blocks all |
| **Saga** | Each service does its local transaction + publishes event; each step has a compensating action | Eventual consistency; complex; handles failures gracefully |

**Saga example for reservation:**
```
1. Reservation Service: create reservation (PENDING) → emit "reservation_created"
2. Inventory Service: decrement inventory → emit "inventory_decremented"
3. Payment Service: charge card → emit "payment_completed"
4. Reservation Service: mark reservation CONFIRMED

If Payment fails:
← Payment Service: emit "payment_failed"
← Inventory Service: increment inventory back (compensating transaction)
← Reservation Service: mark reservation CANCELLED
```

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| Relational DB (MySQL) | ACID transactions prevent double-booking; schema is clearly relational; low write QPS |
| `room_type_inventory` table | Track availability by room type + date; enables overbooking config; simple queries |
| One row per (hotel, room_type, date) | Easy date-range queries; simple availability check; one UPDATE per booking |
| Idempotency key (`reservation_id`) | Prevents double-booking from accidental double-clicks or network retries |
| Optimistic locking | No DB locks; good throughput; sufficient for hotel's low-contention workload |
| DB constraint as alternative | Even simpler; guaranteed at DB level; no application code needed |
| Reservation + Inventory in same service | Single SQL transaction prevents partial state; avoids distributed transaction complexity |
| Redis cache for inventory reads | 300 QPS browsing served from memory; DB only handles actual bookings (3 QPS) |
| Shard by hotel_id | Every query filters by hotel_id; no cross-shard joins needed; linear scaling |

---

## Quick Interview Q&A Cheat Sheet

**Q: How do you prevent two users from booking the last available room simultaneously?**  
A: Use **optimistic locking** — add a `version` column to `room_type_inventory`. The UPDATE includes `AND version = {read_version}`. If 0 rows affected, another user won the race → retry. Alternatively, use a DB CHECK constraint `total_inventory - total_reserved >= 0` — the second update violates the constraint and is rejected automatically.

**Q: What is an idempotency key and why does hotel booking need it?**  
A: Server generates a unique `reservation_id` UUID when user opens the booking page. User submits this ID with their booking. The `reservation_id` has a UNIQUE DB constraint. If user double-clicks or browser retries, the second request with the same ID is rejected by the constraint → only one reservation created.

**Q: Why use one row per date in the inventory table instead of storing start/end date ranges?**  
A: Date-per-row enables simple queries: `WHERE date BETWEEN check_in AND check_out AND (total_reserved + N) <= 110% * total_inventory`. Range-based storage requires complex interval overlap logic. Date-per-row is also easier to UPDATE atomically per night.

**Q: Pessimistic vs. optimistic locking — which for hotel reservations?**  
A: Optimistic locking preferred. Hotel reservations have low-to-medium contention (most room types aren't fully booked most of the time). Optimistic locking allows concurrent reads with no lock overhead; only conflicts on the rare simultaneous booking of the last room. Pessimistic locking serializes all bookings for a room type, hurting throughput unnecessarily.

**Q: How do you handle payment failure after inventory is decremented?**  
A: Two-phase reservation: (1) Decrement inventory + create PENDING reservation in one transaction; (2) Call payment gateway. If payment fails → compensating transaction: increment inventory back, mark reservation CANCELLED. Hold PENDING reservations for 15 minutes max — a background job releases held inventory for unpaid reservations.

**Q: How do you scale the inventory database to handle 1,000× more traffic?**  
A: Shard by `hotel_id` — all queries filter by hotel_id so no cross-shard joins. With 16 shards, a 30,000 QPS system = ~1,875 QPS per shard. Add Redis cache for availability reads (300 QPS browsing vs. 3 QPS actual booking) — most read traffic served from memory without touching the DB.

**Q: How does overbooking work technically?**  
A: The availability check uses `110%` of `total_inventory` as the threshold: `total_reserved + N <= 1.1 × total_inventory`. This allows selling up to 10% more rooms than physically exist. The hotel sets this percentage. If cancellations happen as expected, no guest is turned away. If the hotel is overbooked when guests arrive, they're walked to another hotel (at the hotel's expense).
