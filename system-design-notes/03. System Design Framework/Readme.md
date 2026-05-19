# Chapter 3: A Framework for System Design Interviews

## Introduction
System design interviews are a key part of the hiring process, simulating real-life problem-solving scenarios. These interviews evaluate not just technical skills but also collaboration, communication, and the ability to handle ambiguous requirements.

This chapter introduces a **4-step framework** for navigating system design interviews effectively.

---

## Step 1: Understand the Problem and Establish Design Scope

### Key Objectives
- Clarify requirements and assumptions.
- Avoid jumping into solutions prematurely.
- Showcase critical thinking by asking good questions.

### Approach
- **Ask Clarifying Questions:**
  - What are the most important features?
  - What scale does the system need to handle?
  - Are we building for web, mobile, or both?
  - Are there existing technologies or constraints?

- **Document Assumptions:** Write assumptions on a whiteboard or paper for reference.

### Example
**Problem:** Design a news feed system.  
**Questions:**
- Is it a mobile app, web app, or both?
- How many friends can a user have?
- Should the feed include images and videos?
- Is the feed sorted by reverse chronological order?

---

## Step 2: Propose High-Level Design and Get Buy-In

### Key Objectives
- Develop a high-level architecture.
- Collaborate with the interviewer to refine the design.

### Approach
- **Draft a Blueprint:**
  - Use box diagrams for key components (e.g., clients, APIs, databases, caches, CDNs).
  - Treat the interviewer as a teammate to refine the design.

- **Perform Back-of-the-Envelope Calculations:**
  - Ensure the design can handle the scale constraints.

- **Walk Through Use Cases:** Identify edge cases and validate design assumptions.

### Example
For a news feed system, divide the design into:
1. **Feed Publishing Flow:** Writing posts into databases and populating friends' feeds.
2. **Feed Retrieval Flow:** Aggregating and displaying friends' posts in reverse chronological order.

---

## Step 3: Design Deep Dive

### Key Objectives
- Dive into critical components.
- Showcase depth of understanding and adaptability.

### Approach
- **Prioritize Key Components:** Focus on areas most relevant to the problem.
- **Discuss Bottlenecks:** Identify potential performance issues and propose solutions.
- **Balance Detail:** Avoid over-engineering or unnecessary deep dives.

### Example Topics
- **URL Shortener:** Focus on hash function design.
- **Chat System:** Explore latency reduction and online/offline status handling.
- **News Feed System:** Examine feed publishing and retrieval processes.

---

## Step 4: Wrap-Up

### Key Objectives
- Highlight areas for improvement.
- Recap the design and discuss follow-ups.

### Approach
- **Identify Bottlenecks:** Discuss potential limitations and scaling strategies.
- **Summarize Design:** Recap major design decisions and trade-offs.
- **Propose Enhancements:**
  - How to scale from 1 million to 10 million users.
  - Error handling for server failures or network issues.

---

## Best Practices

### Dos
- **Ask Questions:** Clarify ambiguities before diving into solutions.
- **Communicate:** Share your thought process with the interviewer.
- **Iterate with the Interviewer:** Treat them as a collaborator.
- **Show Flexibility:** Suggest alternative approaches and refine your design.
- **Focus on Critical Components:** Prioritize key parts of the system.

### Don’ts
- **Avoid Premature Solutions:** Don’t design before understanding the requirements.
- **Don’t Go Silent:** Communicate regularly during the process.
- **Avoid Over-Engineering:** Focus on practical, scalable solutions.

---

## Time Management

### Suggested Time Allocation (for 45-Minute Interviews):
1. **Understand Problem and Scope:** 3–10 minutes
2. **High-Level Design and Buy-In:** 10–15 minutes
3. **Deep Dive:** 10–25 minutes
4. **Wrap-Up:** 3–5 minutes

---

## Most Asked Interview Questions

**Q1. What are the 4 key steps in a system design interview?**
> (1) Understand the Problem and Establish Scope (ask clarifying questions, define requirements); (2) Propose High-Level Design and get buy-in (draw boxes, APIs, data flow); (3) Design Deep Dive (focus on critical components, discuss trade-offs); (4) Wrap Up (summarize, bottlenecks, future improvements). This structure shows organized thinking.

**Q2. Why is it critical to clarify requirements before jumping into a design?**
> Without clarification, you may spend 45 minutes designing the wrong system. Different assumptions (scale, features, constraints) lead to drastically different architectures. Asking "how many users are we designing for?" or "is mobile support required?" signals maturity and prevents wasted effort.

**Q3. What is the difference between functional and non-functional requirements?**
> Functional requirements define what the system does (e.g., "users can send messages", "search returns results in 100ms"). Non-functional requirements define how well it does it (e.g., 99.99% availability, <200ms p99 latency, 100M DAU scale, data durability). Both must be established before designing.

**Q4. How do you decide what to prioritize when the problem scope is ambiguous?**
> Ask the interviewer directly: "What are the most important features to get right?" or "Should I focus on the happy path first?". When truly unclear, prioritize: (1) core read/write paths over edge cases, (2) scale constraints that drive architecture choices, (3) unique challenges of the domain (consistency for payments, latency for real-time apps).

**Q5. What questions should you always ask at the start of a system design interview?**
> (1) What is the scale? (DAU, QPS, storage); (2) What are the core features? (don't assume); (3) What are the latency/availability requirements?; (4) Is this a new system or redesign of an existing one?; (5) Any specific technology constraints or preferences?; (6) Where are users located — global or regional?

**Q6. When should you do a deep dive vs. staying at the high level?**
> Spend the first 10–15 minutes on high-level design to get interviewer buy-in on the overall approach. Then ask "Would you like to dive deeper into any specific component?" Deep dive into the most complex or unique parts of the design — the parts where your knowledge differentiation matters most (e.g., the storage system, the fanout mechanism, the consensus protocol).

**Q7. How do you communicate trade-offs during a system design interview?**
> Always frame decisions as trade-offs, not absolutes: "I'm choosing SQL over NoSQL here because we need ACID transactions for payment, but this limits our write scalability — we can address that later with sharding." Explicitly naming the alternative option and why you didn't choose it demonstrates senior engineering judgment.

**Q8. What are the most common mistakes candidates make?**
> (1) Jumping straight into implementation without requirements; (2) Over-engineering from the start; (3) Choosing technologies without justification; (4) Not managing time (spending 30 min on one component); (5) Silence — not narrating thought process; (6) Using buzzwords without understanding (e.g., saying "Kafka" without knowing why); (7) Ignoring non-functional requirements.

**Q9. How should you handle a component you don't know well?**
> Be honest but constructive: "I'm less familiar with X — I'd likely choose Y which I know achieves similar goals, though X may have advantages I'd research." Interviewers value honesty over bluffing. Don't invent technical details — a wrong assumption about a technology can undermine your entire design.

**Q10. How do you estimate scale requirements during the scoping phase?**
> Use structured decomposition: DAU → requests per user per day → average QPS → peak QPS (×3). For storage: writes per day × average object size × retention period. Always state your assumptions out loud. Round numbers to make math easy; precision isn't the goal, order of magnitude is.

**Q11. What does a good system design drawing include?**
> Client(s), load balancer, API servers, cache layer, primary database, any secondary databases or data stores, message queues, background workers, CDN for static content, and monitoring/logging. Label data flows with arrows showing direction. Keep it legible — boxes and lines are enough.

**Q12. How do you handle unavoidable bottlenecks in a system design?**
> Identify them explicitly: "The database is likely our bottleneck at this scale." Then propose mitigations: caching to reduce DB reads, read replicas, sharding for write scalability. Show awareness of where the next bottleneck arises after each mitigation. This demonstrates systems-thinking.

**Q13. What is the API design process in a system design interview?**
> Define the key APIs for your system: HTTP method (GET/POST/PUT/DELETE), endpoint path, request parameters, response structure, and authentication mechanism. Think about idempotency for mutations. Mention rate limiting and pagination for list APIs. You don't need to spec every field — focus on the most important APIs to demonstrate reasoning.

**Q14. When should you use REST vs. GraphQL vs. gRPC?**
> REST: standard, widely understood, good for public APIs; GraphQL: when clients need flexible query shapes or multiple resources in one request (e.g., mobile apps with bandwidth constraints); gRPC: low-latency internal service-to-service communication, schema-enforced via protobuf, excellent for microservices. Choose based on who consumes the API and what performance characteristics matter.

**Q15. How do you approach data model design in a system design interview?**
> Start with the entities (User, Post, Comment, etc.), define their key attributes, then decide on primary keys and relationships. Ask: "Is this query pattern read-heavy or write-heavy?" to guide normalization vs. denormalization. Consider how the data access pattern (by user ID, by time range, by geographic area) drives index and sharding key selection.

**Q16. How do you address consistency, availability, and partition tolerance in your design?**
> Explicitly state your consistency model choice and justify it: "For user profile reads, eventual consistency is acceptable — I'll use DynamoDB (AP system) with TTL-based cache. For financial transactions, I need strong consistency — I'll use PostgreSQL with ACID guarantees." Show awareness of the trade-off, not just blind technology selection.

**Q17. What is the wrap-up phase of a system design interview, and what should you cover?**
> In the final 3–5 minutes: (1) Briefly summarize the overall design; (2) Identify bottlenecks or areas you'd improve given more time; (3) Mention monitoring/alerting strategy; (4) Address failure scenarios (what if the DB goes down?); (5) Discuss how you'd handle future scale increases (10× growth). This demonstrates operational maturity.

**Q18. How do you handle a totally unfamiliar system design problem?**
> Apply first principles: (1) What data needs to be stored?; (2) How is data written and read?; (3) What are the scale and latency requirements?; (4) What are the failure scenarios? Even an unfamiliar domain has familiar patterns — most systems are some combination of read/write APIs, storage, caching, queues, and compute. Map the problem to patterns you know.

**Q19. Should you use microservices or a monolith in a system design interview?**
> Start with a monolith-like high-level design (fewer boxes = clearer communication) then identify which components would be extracted as separate services if the scale demands it. Jumping straight to 15 microservices makes the diagram noisy. Show you understand the trade-off: "In production, I'd likely break this into separate services as the team and load grow."

**Q20. How do you address security in a system design interview?**
> Mention it proactively but briefly: authentication (JWT, OAuth), authorization (RBAC), data encryption (TLS in transit, AES at rest), rate limiting (prevent DoS), input validation, and audit logging. Unless security is the focus of the problem, don't spend more than 2–3 minutes on it — but not mentioning it at all is a red flag for senior roles.

**Q21. How do you design for high availability in a system design interview?**
> High availability means eliminating single points of failure. Key patterns: (1) Load balancers in front of stateless services; (2) Database replication with automatic failover; (3) Multi-AZ or multi-region deployment; (4) Health checks and automatic instance replacement; (5) Circuit breakers for downstream services; (6) Graceful degradation when dependencies fail.

**Q22. How do you demonstrate scalability in your design?**
> Show that each component can scale independently: "The API servers are stateless — I can add more with a load balancer. The message queue buffers write bursts. The database uses read replicas for reads and sharding for write scalability." Explicitly address the path from current scale to 10× growth.

**Q23. What is the difference between a system design interview at different seniority levels?**
> Junior: expected to design basic systems with guidance, focus on component knowledge. Senior: expected to define scope, identify bottlenecks, make trade-off calls, consider operational concerns (monitoring, failure handling). Staff/Principal: expected to identify ambiguity in requirements, propose multiple architectures, reason about cost and organizational impact.

**Q24. What role does monitoring and observability play in system design?**
> A well-designed system includes: metrics (QPS, latency, error rate, saturation), logs (structured logs for debugging), and traces (distributed tracing for request flows). Mention Prometheus + Grafana for metrics, ELK/Splunk for logs, Jaeger/Zipkin for tracing. SLOs (Service Level Objectives) should guide alerting thresholds.

**Q25. How do you handle data migrations in a large-scale system?**
> Large migrations must be done without downtime: (1) Write to both old and new tables (dual-write); (2) Backfill old data in batches; (3) Validate consistency between old and new; (4) Switch reads to new store; (5) Remove old dual-write. This "expand-migrate-contract" pattern ensures zero downtime. Shadow reads and feature flags help control rollout safely.

**Q26. What is an SLA vs. SLO vs. SLI?**
> SLI (Service Level Indicator) is the actual measured metric (e.g., "99.95% of requests returned in <200ms"). SLO (Service Level Objective) is the target for the SLI (e.g., "99.9% of requests must complete in <200ms"). SLA (Service Level Agreement) is a contractual commitment with customers/consequences for breach. Design decisions should be driven by SLOs.

**Q27. How do you design a system that needs to handle a sudden 10× traffic spike?**
> (1) Auto-scaling groups for compute (scale out within minutes); (2) Message queues to absorb burst writes and process asynchronously; (3) Caching to absorb burst reads; (4) Rate limiting to protect downstream services; (5) CDN for static content (no origin hit); (6) Load shedding — gracefully reject excess requests with 429 instead of crashing under overload.

