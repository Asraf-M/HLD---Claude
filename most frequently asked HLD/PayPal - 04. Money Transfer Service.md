# System Design: Money Transfer Service (PayPal-Style P2P) (Beginner-Friendly Guide)

---

## What Are We Building?

A peer-to-peer money transfer system where users can send money to each other instantly (or within hours for bank transfers):
- User A sends $50 to User B's email address
- User B receives notification: "You received $50 from A"
- Money instantly available in B's wallet
- Users can transfer to friends, family, or merchants
- Support domestic (same country) and international transfers
- Instant transfers (PayPal) vs slow transfers (bank ACH)
- Handle requests to pay (User A sends B a "request" for $20)

Think: PayPal.me, Venmo, or Square Cash—"send money easily."

**Key Engineering Challenges:**
- **Instant settlement** — User sends $50; receiver sees it in wallet immediately (but bank doesn't confirm for 2-3 days)
- **Cross-currency transfers** — User in US sends $100 to friend in India (INR); handle conversion, rates, fees
- **Duplicate detection** — User hits "send" twice; system must charge only once (idempotency)
- **Mobile reliability** — App on 3G network; request times out; user retries; prevent double-charge
- **Fraud at scale** — $1B/day in transfers; 0.1% fraud = $1M loss/day; real-time detection critical
- **Receiver doesn't need account** — Can receive money before signing up; account created on first login

---

## Step 1: Design Scope

**Scale:**
| Parameter | Value |
|-----------|-------|
| Daily transfers | 500+ million |
| Transfers/second (peak) | 100,000 |
| Daily transfer volume | $50+ billion |
| Average transfer size | $100 |
| Domestic (same country) | 80% |
| International | 20% |
| Instant transfers | 70% |
| Bank transfers (slow) | 30% |
| User base | 300+ million |
| Success rate | 99%+ |
| Fraud loss rate | < 0.05% |

**QPS Funnel:**
```
Transfer requests:        100,000 QPS
Instant transfers:         70,000 QPS
Bank transfers (enqueued):  30,000 QPS
Transfer status checks:    500,000 QPS
```

**Non-functional requirements:**
- Availability: 99.99% uptime
- Transfer latency: Instant (< 5 seconds), Bank (1-3 business days)
- Fraud detection latency: < 100ms (real-time)
- Consistency: Strong (no double-transfers)
- Scalability: Handle 2x traffic spike

---

## Step 2: API Design

**Transfer APIs:**

```
POST   /v1/transfers                ← Initiate transfer
GET    /v1/transfers/{id}           ← Check status
POST   /v1/transfers/{id}/cancel    ← Cancel pending
POST   /v1/transfers/{id}/retry     ← Retry failed
POST   /v1/transfer-requests        ← Request money from someone
POST   /v1/transfer-requests/{id}/approve ← Approve request
```

**Example: Send Money (Instant)**
```json
POST /v1/transfers
{
  "recipient": {
    "type": "email",  // or "phone", "user_id"
    "value": "friend@example.com"
  },
  "amount": 50.00,
  "currency": "USD",
  "transfer_type": "INSTANT",  // or "BANK"
  
  "note": "Dinner split",
  "idempotency_key": "transfer_12345_attempt_1"
}

Response:
{
  "transfer_id": "xfer_abc123",
  "status": "COMPLETED",  // INSTANT transfers are fast
  "sender_id": "user_123",
  "recipient_identifier": "friend@example.com",
  "amount": 50.00,
  "fee": 0.00,  // No fees for personal transfers
  "completed_at": "2026-06-18T10:35:00Z",
  
  "recipient_user_id": "user_456"  // Resolved from email
}
```

**Example: Request Money**
```json
POST /v1/transfer-requests
{
  "requester_id": "user_123",  // Who's asking
  "recipient": {
    "type": "email",
    "value": "debtor@example.com"
  },
  "amount": 75.00,
  "currency": "USD",
  "note": "Concert tickets",
  
  "idempotency_key": "request_6789_1"
}

Response:
{
  "request_id": "treq_xyz789",
  "status": "PENDING",
  "requester_id": "user_123",
  "recipient_email": "debtor@example.com",
  "amount": 75.00,
  "expires_at": "2026-07-18T10:35:00Z"  // 30 days to pay
}
```

**Example: Bank Transfer (Slow)**
```json
POST /v1/transfers
{
  "recipient": {
    "type": "bank_account",
    "bank_account": {
      "account_number": "1234567890",
      "routing_number": "021000021",  // Wells Fargo
      "account_holder": "Jane Doe"
    }
  },
  "amount": 500.00,
  "currency": "USD",
  "transfer_type": "BANK",  // Slow (1-3 days)
  
  "idempotency_key": "bank_withdraw_999_1"
}

Response:
{
  "transfer_id": "xfer_def456",
  "status": "PENDING",
  "estimated_arrival": "2026-06-20T10:35:00Z",  // 2-3 days
  "fee": 0.00
}
```

---

## Step 3: Database Design

**Why Multiple Databases?**

| Data | Database | Why? |
|------|----------|------|
| Transfer records | PostgreSQL | ACID, auditability, queryable |
| Transfer status (in-flight) | Redis | Fast updates, expires |
| Batch queue (bank transfers) | Kafka | Reliable queuing, async processing |
| Risk scoring | Redis + ML | Real-time fraud detection |
| Exchange rates | PostgreSQL + Redis | Historical rates, caching |
| Settlement ledger | PostgreSQL | Double-entry accounting |

---

## Step 4: Data Schema

**Transfers Table (PostgreSQL):**
```sql
CREATE TABLE transfers (
  transfer_id VARCHAR PRIMARY KEY,
  
  sender_id VARCHAR NOT NULL,
  recipient_identifier VARCHAR,  -- email, phone, user_id
  recipient_user_id VARCHAR,  -- resolved after lookup
  
  amount DECIMAL(15,2),
  currency VARCHAR(3),
  
  transfer_type VARCHAR,  -- INSTANT, BANK, INTERNATIONAL
  status VARCHAR(20),  -- INITIATED, PENDING, COMPLETED, FAILED, CANCELLED
  
  fee DECIMAL(15,2),  -- Charged to sender
  exchange_rate DECIMAL(10,6),  -- For international
  recipient_receives DECIMAL(15,2),  -- After fees & conversion
  
  initiated_at TIMESTAMP,
  completed_at TIMESTAMP,
  
  idempotency_key VARCHAR UNIQUE,
  note VARCHAR,
  
  fraud_risk_score INT,  -- 0-100
  fraud_status VARCHAR,  -- APPROVED, FLAGGED, BLOCKED
  
  settlement_batch_id VARCHAR,  -- For batch settlement
  
  INDEX (sender_id, initiated_at),
  INDEX (recipient_user_id, initiated_at),
  INDEX (status),
  INDEX (idempotency_key)
);
```

**Transfer Requests Table:**
```sql
CREATE TABLE transfer_requests (
  request_id VARCHAR PRIMARY KEY,
  
  requester_id VARCHAR NOT NULL,  -- Who's asking
  recipient_identifier VARCHAR,  -- email, phone
  recipient_user_id VARCHAR,  -- resolved
  
  amount DECIMAL(15,2),
  currency VARCHAR(3),
  
  status VARCHAR(20),  -- PENDING, APPROVED, REJECTED, EXPIRED
  
  created_at TIMESTAMP,
  expires_at TIMESTAMP,  -- 30 days from creation
  
  note VARCHAR,
  
  INDEX (requester_id, created_at),
  INDEX (recipient_user_id, created_at),
  INDEX (status, expires_at)
);
```

**Exchange Rate Cache (Redis):**
```json
Key: exchange_rate:USD:INR:2026-06-18
Value: {
  "from": "USD",
  "to": "INR",
  "rate": 83.45,
  "timestamp": 1718704500000,
  "source": "XE API"
}
TTL: 1 hour
```

---

## Step 5: High-Level Architecture

```
┌──────────────────────────┐
│  Mobile/Web Apps         │
└──────────┬───────────────┘
           │
    ┌──────▼──────────────┐
    │  API Gateway        │
    │  (Auth, Rate Limit) │
    └──────┬──────────────┘
           │
    ┌──────┴────────────────────────┐
    │                               │
┌───▼──────────────┐  ┌────────────┐ │
│Transfer Service  │  │Fraud       │ │
│                  │  │Detection   │ │
└───┬──────────────┘  └────┬───────┘ │
    │                      │         │
    ├──────┬───────────────┤         │
    │      │               │         │
┌───▼──┐ ┌─▼──┐ ┌────────┐ │ ┌──────▼───┐
│Wallet│ │PG  │ │Kafka   │ │ │Settlement│
│Svc   │ │DB  │ │Queue   │ │ │Service   │
└──────┘ └────┘ └────────┘ │ └──────────┘
    │                       │
    └───────────────────────┘
           │
    ┌──────▼──────────────┐
    │ Bank Integration    │
    │ (ACH, SWIFT)        │
    └─────────────────────┘

Services:
- Transfer Service: Initiates transfers, handles logic
- Fraud Detection: Real-time risk scoring
- Settlement Service: Batch processing to banks
- Bank Integration: Routes to ACH/SWIFT networks
```

---

## Step 6: Instant vs Bank Transfers

**Instant Transfer (P2P):**
```
User A sends $50 to User B instantly
       ↓
1. Check A's balance: $100 >= $50? Yes
2. Deduct from A: balance = $50
3. Add to B: balance = $300 (was $250)
4. Record in ledger (both sides)
5. Send notification to B
6. Return success to A

Result: COMPLETED in < 5 seconds
No bank involvement, PayPal just moves money between wallets
```

**Bank Transfer (Slow):**
```
User C withdraws $1000 to bank account
       ↓
1. Check C's balance: $2000 >= $1000? Yes
2. Deduct from C: balance = $1000 (wallet reduced)
3. Create ACH batch entry
4. Enqueue to Kafka: "ACH outbound $1000"
5. Return status: PENDING

Next morning (batch processing):
6. Batch 1000+ withdrawal requests
7. Send ACH file to bank: "Transfer $1M to account 123"
8. Bank processes over 1-3 days
9. Receive confirmation: "ACH settled"
10. Update status: COMPLETED

Result: Completes in 2-3 business days (regulated process)
```

---

## Step 7: Recipient Resolution & Account Creation

**Problem:** User sends $50 to "friend@example.com". Friend doesn't have PayPal yet.

**Solution: Implicit Account Creation**

```
Step 1: Send to email
POST /v1/transfers
{
  "recipient": {"type": "email", "value": "friend@example.com"},
  "amount": 50.00
}

Step 2: Recipient lookup
- Query user database: "friend@example.com" = user_456?
- Not found: Create placeholder account
  - Account status: UNCLAIMED
  - Email: friend@example.com
  - Balance: $50 (reserved)

Step 3: Send notification
- Email to friend@example.com: "You received $50! Claim it here"
- Link: /claim-payment?code=xyz

Step 4: Friend claims account
- Clicks link
- Creates password
- Account status: ACTIVE
- Balance: $50 (now accessible)

Result: Friend can now use PayPal, send/receive, withdraw
```

---

## Step 8: Fraud Detection in Transfers

**Real-Time Risk Scoring:**

```
User initiates transfer: $5,000 to new recipient

Risk factors (ML model):
- New recipient? (Never transferred before) → +20 points
- Amount > typical? (User usually sends $100, now $5K) → +25 points
- Time unusual? (Transfer at 3am, user usually 9-5) → +10 points
- Location unusual? (Recipient in different country) → +15 points
- Velocity high? (5 transfers in 10 minutes) → +30 points

Total risk score: 100 (MAX) → BLOCK transfer

Actions:
1. Block transfer
2. Send email: "Unusual activity, verify this transfer"
3. Send SMS: "Confirm transfer via code"
4. User confirms: Score reduced, transfer proceeds
5. If not confirmed: Transfer stays blocked (customer calls support)
```

**Model Training:**
```
Historical data: 1M fraudulent transfers + 100M legitimate
Features:
- Recipient (new vs known)
- Amount (outlier vs typical)
- Time (unusual hour)
- Location
- Velocity (transfers per hour)
- Device (new phone)
- IP (VPN, proxy, suspicious)

Train daily on: Confirmed fraud cases
Goal: Catch 99% of fraud, false positive rate < 1%
```

---

## Step 9: Key Design Decisions & Tradeoffs

| Decision | Why? | Tradeoff |
|----------|------|----------|
| Instant for P2P transfers | User expectation; money shows immediately | Risk window (fraud can happen before settlement) |
| Bank transfers slow (2-3 days) | Regulatory requirement; banking network limitation | Users frustrate; support calls |
| Implicit account creation | Frictionless; no signup needed to receive | Security risk (phishing to wrong email); account takeover |
| Idempotency keys | Prevent duplicate transfers on retry | Extra DB lookup; cache overhead |
| Real-time fraud detection | Catch fraud early; reduce loss | False positives block legit users; support burden |
| Multiple payment routes | Reliability; if ACH fails, use wire | Higher fee structure; complexity |

---

## Step 10: Interview Cheat Sheet Q&A

**Q: User sends $100 to email address, recipient hasn't signed up yet. How is money stored?**  
A: Placeholder account created automatically with status "UNCLAIMED". Balance reserved ($100 marked as unclaimed). Email sent to recipient with claim link. When recipient creates account (verifies email), account status changes to "ACTIVE" and balance becomes accessible. If unclaimed for 180 days, auto-refund to sender.

**Q: What if user hits "send" twice in quick succession (network retry)? Do they lose $200?**  
A: No, idempotency keys prevent it. Client generates unique key per transfer (transfer_123_attempt_1). Server checks: if key exists, return cached result (don't process again). First "send": success, key stored. Second "send": key found, return cached success, no charge. Only one $100 transferred.

**Q: How do we handle international transfers (USD to INR)? Who decides the exchange rate?**  
A: Rates updated hourly from XE or similar APIs. When transfer initiated, rate locked (rate at time of transfer, not settlement). PayPal also charges 1.5% markup. User sees: "$100 USD = 8,345 INR at 83.45 rate + 1.5% fee = recipient gets 8,218 INR". Customer sees total cost upfront, no surprises.

**Q: Transfer fails (bank says "account closed"). How do we reverse it?**  
A: Reverse the ledger entry: add money back to sender's account. Create refund transfer_id. Send notification: "Transfer failed, money refunded." If sender doesn't see refund within 5 minutes, manual support intervention. Ledger preserves both original and reversal entries (audit trail).

**Q: Fraud system flags transfer as suspicious. Should we block or ask user?**  
A: Depends on risk. Low-risk: let through (fraud score < 50). Medium risk (50-75): send SMS verification ("Confirm transfer with code sent to your phone"). High risk (> 75): block, require call to support. This balances security (catch fraud) with UX (don't block legit transactions).

**Q: What if recipient's bank account is invalid? How do we detect?**  
A: During bank transfer setup, micro-verify: send two small deposits ($0.10, $0.05). Recipient confirms amounts in their bank statement. If they can't confirm, account invalid. For large transfers, mandate this verification. Prevents fraud (wrong account) and money loss.

---

## Summary

A money transfer service requires:
- ✅ Instant P2P transfers (< 5 seconds)
- ✅ Bank transfers (ACH/SWIFT, 2-3 days)
- ✅ Idempotency keys (prevent duplicates)
- ✅ Recipient resolution (email, phone, user_id)
- ✅ Implicit account creation (unclaimed accounts)
- ✅ Real-time fraud detection (99% catch rate)
- ✅ Exchange rate handling (international transfers)
- ✅ Batch settlement (queue + processing)
- ✅ Ledger with reversal support
- ✅ High reliability (99.99% uptime, strong consistency)
