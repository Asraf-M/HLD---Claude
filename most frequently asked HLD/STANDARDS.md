# HLD Documentation Standards & Format Guide

**Last Updated:** 2026-06-18  
**Applied to:** All 14 "most frequently asked HLD" files  
**Reference Template:** Hotel Reservation System README.md

---

## Overview

This document standardizes the format for all High-Level Design (HLD) documentation. All files must follow this structure to maintain consistency, readability, and interview-readiness.

---

## Complete File Structure (Mandatory Sections)

Every HLD file MUST include these 10 sections in this exact order:

### 1. Title
```markdown
# System Design: [System Name] (Beginner-Friendly Guide)
```
- Use this exact format for all titles
- "Beginner-Friendly Guide" signals the tone is educational
- Examples: "System Design: Ride-Sharing Backend (Beginner-Friendly Guide)"

---

### 2. What Are We Building?
**Purpose:** Business context + engineering challenges  
**Length:** 150-250 words  
**Structure:**
- Opening paragraph: What is this system? What does it do?
- 5-6 key engineering challenges, each **bolded**
- Each challenge has a 1-line explanation of why it's hard

**Template:**
```markdown
## What Are We Building?

A [system description] where:
- Feature 1
- Feature 2
- Feature 3

**Key Engineering Challenges:**
- **[Challenge 1]** — [why it's hard]
- **[Challenge 2]** — [why it's hard]
- **[Challenge 3]** — [why it's hard]
- **[Challenge 4]** — [why it's hard]
- **[Challenge 5]** — [why it's hard]
- **[Challenge 6]** — [why it's hard]
```

**Example challenges:**
- Real-time matching at scale
- High durability (11 nines)
- Consistency across regions
- Cost optimization
- Geospatial indexing
- Event-driven architecture

---

### 3. Design Scope — Scale & Requirements
**Purpose:** Quantify the problem  
**Must Include:**

**a) Scale Table:**
```markdown
| Parameter | Value |
|-----------|-------|
| Daily active users | 10 million |
| Concurrent users | 1 million (peak) |
| Requests/second | 100,000 QPS |
| Data storage | 1+ petabytes |
| ... | ... |
```

Key metrics to include (adjust per system):
- Daily/monthly active users
- Concurrent users at peak
- Requests per second (QPS)
- Data volume
- Geographic regions
- Availability target
- Latency target

**b) QPS Funnel:**
Show how traffic breaks down by operation type:
```markdown
**QPS Funnel:**
```
Total requests:       500,000 QPS
Read requests:        400,000 QPS (80%)
Write requests:       100,000 QPS (20%)
```
```

**c) Non-Functional Requirements:**
- Availability (99.9%, 99.99%, etc.)
- Latency targets (p50, p99)
- Consistency model
- Scalability expectations
- Security requirements

---

### 4. API Design
**Purpose:** Specify how clients interact with the system  
**Must Include:**

**a) Endpoint Listing:**
```markdown
**Core Endpoints:**
```
POST   /resource              ← Create
GET    /resource/{id}         ← Read
PUT    /resource/{id}         ← Update
DELETE /resource/{id}         ← Delete
```
```

**b) Request/Response Examples:**
For every major endpoint, include:
```json
POST /rides/request
{
  "pickup_latitude": 37.7749,
  "pickup_longitude": -122.4194,
  "dropoff_latitude": 37.8044,
  "dropoff_longitude": -122.2712
}

Response:
{
  "ride_id": "ride_123456",
  "status": "MATCHING",
  "estimated_arrival_time": "4 minutes"
}
```

**Guidelines:**
- Show 2-4 major endpoints with examples
- Include error cases if relevant
- Use realistic data in examples
- Include HTTP method and expected response code

---

### 5. Database Design
**Purpose:** Justify technology choices  
**Must Include:**

**a) Decision Table:**
```markdown
| Use Case | Database | Why? |
|----------|----------|------|
| User profiles | PostgreSQL | Structured, ACID |
| Real-time data | Redis | Sub-millisecond latency |
| Historical logs | Elasticsearch | Full-text search |
| Analytics | BigQuery | OLAP queries |
```

**Technology comparison criteria:**
- Structured vs unstructured data
- Latency requirements
- Consistency requirements (ACID vs eventual)
- Query patterns
- Scalability needs
- Cost considerations

---

### 6. Data Schema
**Purpose:** Show how data is organized  
**Include Both SQL and NoSQL Examples:**

**SQL Schema Example:**
```sql
CREATE TABLE users (
  user_id VARCHAR PRIMARY KEY,
  name VARCHAR,
  email VARCHAR UNIQUE,
  rating DECIMAL(3,1),
  created_at TIMESTAMP
);
```

**NoSQL Example (if applicable):**
```json
{
  "user_id": "user_123",
  "profile": {
    "name": "John Doe",
    "email": "john@example.com",
    "rating": 4.8
  },
  "metadata": {
    "created_at": "2024-06-18",
    "last_login": "2024-06-18T10:35:00Z"
  }
}
```

**Guidelines:**
- Show 2-3 primary tables/collections
- Explain key design decisions
- Include important indexes
- Mark primary/foreign keys clearly
- Add brief comments explaining fields

---

### 7. High-Level Architecture
**Purpose:** Visual overview of system components  
**Must Include:**

**a) ASCII Diagram:**
```markdown
┌─────────────────────┐
│  Client Apps        │
└──────────┬──────────┘
           │
    ┌──────▼──────────┐
    │  API Gateway    │
    └──────┬──────────┘
           │
    ┌──────┴──────┐
    │             │
┌───▼──────┐ ┌───▼──────┐
│Service A  │ │Service B  │
└──────────┘ └──────────┘
```

**b) Component Description Table:**
```markdown
| Component | Responsibility | Technology |
|-----------|---|---|
| API Gateway | Route requests, auth | Nginx, Kong |
| Service A | Core logic | Java, Python |
| Database | Persistence | PostgreSQL |
| Cache | Speed up reads | Redis |
```

**Guidelines:**
- Use ASCII diagrams (box drawing characters)
- Show 5-8 key components
- Show data flow between components
- Include client layer, API layer, service layer, data layer
- Add technology choices for each component

---

### 8-N. Deep Dives (2-5 sections depending on system)
**Purpose:** Technical deep dives into system-specific challenges  
**Section Count:** 2-5 sections (adjust per system complexity)

**Common Deep Dive Topics:**
- Specific algorithm (matching, recommendation, routing)
- Handling high scale (sharding, partitioning)
- Real-time features (WebSocket, streaming)
- Consistency patterns (eventual, strong)
- Failure handling (circuit breaker, retries)
- Caching strategies
- Replication and durability
- Monitoring and observability

**Each Deep Dive Should:**
- Have clear problem statement
- Show the solution with code/pseudocode if helpful
- Explain tradeoffs
- Include examples with numbers

---

### 9. Key Design Decisions & Tradeoffs
**Purpose:** Explain what was sacrificed for what benefit  
**Format (Required):**

```markdown
## Step N: Key Design Decisions & Tradeoffs

| Decision | Why? | Tradeoff |
|----------|------|----------|
| [Decision 1] | [Business/technical reason] | [What we gave up] |
| [Decision 2] | [Business/technical reason] | [What we gave up] |
| [Decision 3] | [Business/technical reason] | [What we gave up] |
```

**Example:**
| Decision | Why? | Tradeoff |
|----------|------|----------|
| Use eventually consistent replicas | Higher availability; no coordination overhead | Brief stale reads; harder to debug |
| Multipart uploads | Resume from failure; parallel uploads = 4-8x faster | More complex client logic |
| Surge pricing during peaks | Attracts drivers; balances supply/demand | Poor users priced out; social inequality |

**Guidelines:**
- Include 4-6 major design decisions
- Be honest about tradeoffs (no perfect solutions)
- Explain business impact, not just technical impact
- Helps interviewers understand your thinking

---

### 10. Interview Cheat Sheet Q&A (REQUIRED)
**Purpose:** Common interview questions + expert answers  
**Format (MANDATORY):**

```markdown
## Interview Cheat Sheet Q&A

**Q: [Common interview question]?**  
A: [Expert explanation. 2-3 sentences with concrete reasoning.]

**Q: [Second question]?**  
A: [Explanation]
```

**MUST Include Exactly 6 Q&A Pairs:**

1. **Q1:** Why [design decision 1]? (vs alternative)
2. **Q2:** What happens when [failure scenario]?
3. **Q3:** Can we do [optimization]? Why/why not?
4. **Q4:** How to prevent [attack/problem]?
5. **Q5:** What if [scale changes]?
6. **Q6:** How to debug [common issue]?

**Example Pattern:**
```markdown
**Q: Why use replication instead of single database?**  
A: If the single database goes down, all data is lost. Replication stores copies in different data centers. If DC1 burns down, DC2 still has data. Cost: 2x storage. Benefit: 99.99% availability.

**Q: What if network splits (DC1 ↔ DC2 disconnected)?**  
A: Without replication: Both sides have old data. With replication: Pick one as "primary" (DC1), DC2 goes read-only. When network heals, sync updates. Users see slight lag, but data safe.
```

**Guidelines:**
- 6 Q&A pairs, no more, no less
- Answer should be 2-3 sentences with reasoning
- Include numbers/examples where possible
- Cover failure scenarios, alternatives, optimizations
- Answers should be something an interviewer would ask

---

### 11. Summary
**Purpose:** Recap key requirements  
**Format:**

```markdown
## Summary

A [system] requires:
- ✅ [Key requirement 1]
- ✅ [Key requirement 2]
- ✅ [Key requirement 3]
...
```

**Example:**
```markdown
## Summary

A ride-sharing system requires:
- ✅ Real-time geospatial indexing (Redis Geo or similar)
- ✅ Efficient matching algorithm with scoring
- ✅ Dynamic surge pricing for supply/demand balance
- ✅ Accurate ETA calculation using traffic data
- ✅ Real-time communication (WebSocket)
- ✅ Reliable payment processing
- ✅ Two-sided rating system
```

**Guidelines:**
- 8-12 bullet points
- Each starts with ✅ checkbox
- Short, punchy language
- Should match the major challenges from "What Are We Building?"

---

## Tone & Style Guidelines

### Language
- **Target audience:** Software engineers preparing for system design interviews
- **Tone:** Friendly, educational, non-condescending
- **Explanation level:** Assume reader knows databases/APIs, but not this specific system

### Structure & Readability
- Use markdown headers (`#`, `##`, `###`) for clear hierarchy
- Use tables for comparisons (databases, tradeoffs, decisions)
- Use code blocks for schema, examples, pseudocode
- Use bullet points for lists (not paragraphs)
- Use bold (`**text**`) for important concepts
- Break up long text with subheadings

### Numbers & Examples
- Always include concrete numbers (QPS, latency, storage)
- Use real-world examples (Netflix, Uber, Amazon)
- Explain "why" not just "what"
- Show scale impact: "1M users = N petabytes = Y racks"

### Code Examples
- Use realistic, runnable examples
- Include request/response in JSON
- Use SQL for relational databases
- Use Python/pseudocode for algorithms
- Keep examples short (< 20 lines typical)

---

## Common Mistakes to Avoid

❌ **Don't:**
- Use overly technical jargon without explanation
- Provide false information for the sake of detail
- Make design decisions without explaining tradeoffs
- Forget the Interview Q&A section
- Use vague metrics ("high performance", "scalable")
- Include implementation details (for HLD, focus on architecture)
- Create diagrams that are hard to read
- Skip the Database Design section

✅ **Do:**
- Explain "why" for every design decision
- Include concrete numbers (QPS, latency, storage)
- Show both what works AND what doesn't
- Anticipate common interview follow-ups
- Use ASCII art (simple, readable)
- Balance breadth (all major topics) with depth (understand each)
- Include real-world tradeoffs
- Keep language conversational ("we" language works well)

---

## Section Checklist

Before considering a file complete:

- [ ] Title matches format: "System Design: [Name] (Beginner-Friendly Guide)"
- [ ] "What Are We Building?" has 5-6 bolded challenges
- [ ] Design Scope has scale table, QPS funnel, non-functional requirements
- [ ] API Design has 2-4 endpoints with request/response examples
- [ ] Database Design has decision table (SQL vs NoSQL reasoning)
- [ ] Data Schema shows 2-3 tables/collections with key fields
- [ ] Architecture has ASCII diagram + component table
- [ ] 2-5 Deep Dive sections covering system-specific challenges
- [ ] Key Design Decisions & Tradeoffs table (4-6 rows)
- [ ] Interview Cheat Sheet has **exactly 6 Q&A pairs**
- [ ] Summary has 8-12 bullet points with ✅ checkboxes
- [ ] All sections use consistent markdown formatting
- [ ] No implementation code (this is HLD, not LLD)
- [ ] All diagrams use ASCII art (no external image files)

---

## How to Apply This Standard to New Files

### For Creating a New HLD File:

1. **Copy the structure** — Use one of the 14 completed files as a template
2. **Fill in each section** — Work through sections 1-11 in order
3. **Add concrete numbers** — Research or estimate QPS, storage, scale
4. **Create the architecture diagram** — Draw with ASCII art
5. **Write the deep dives** — Cover 2-5 system-specific challenges
6. **Add tradeoff table** — Explain 4-6 major design decisions
7. **Write 6 Q&A pairs** — Include common interview questions
8. **Run the checklist** — Verify all sections are complete
9. **Proofread** — Check tone, numbers, explanations

### For Updating an Existing File:

1. Check against the checklist above
2. Add any missing sections
3. Ensure Interview Q&A has exactly 6 pairs
4. Verify scale metrics are realistic
5. Update architecture if system has changed

---

## Examples

**Completed exemplars (use as reference):**
- Hotel Reservation System (in claude notes folder) — Most comprehensive reference
- File Storage System — Good for storage/durability concepts
- Ride-Sharing Backend — Good for real-time/geospatial examples
- API Gateway — Good for architecture/routing examples
- Payment Processing System — Good for financial/idempotency concepts

---

## Questions?

If creating a new HLD file and uncertain:
1. Check these standards
2. Compare with one of the 14 completed files
3. Match the structure, tone, and section count
4. Ensure Interview Q&A has exactly 6 pairs
5. Verify all sections are filled in

**Goal:** Every HLD file in this folder should be interview-ready, beginner-friendly, and comprehensive.

