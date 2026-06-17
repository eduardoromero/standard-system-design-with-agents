# Systems Design Pedagogical Guide (Staff Engineer Edition)

**Owner:** Engineering Leadership  
**Audience:** All Engineering Levels  
**Goal:** To force "thinking before coding" by training engineers on the questions they must ask.

---

## What This Document Is

This document is the **pedagogical companion** to the `systems-design-specification-standard.md`. 
It serves as an educational resource for teams learning to think systematically about systems. It walks through the various phases of the system design lifecycle, explaining *why* we ask for specific details and providing "Deep Dives" and "Staff Challenge Questions" to help engineers learn how to do proper systems design.

The best engineers don't jump to solutions. They enumerate constraints, access patterns, and failure modes *first*. This document trains that discipline.

---

## Phase 1: Structural Decomposition (C4)

Before we code, we map the territory. This phase defines boundaries, which dictates security, deployment strategy, and team ownership.

### 1.1. System Context (Level 1)

**The Concept:** The "Context" is the 50,000-foot view. It shows your system as a single black box in the center. Everything else is an "External Actor."

**What You Document:**

- **Users:** All distinct user roles (e.g., Admin, End-User, Anonymous Visitor)
- **External Systems:** Any system you call but don't control (e.g., Stripe, Auth0, Legacy Mainframe, AWS S3)
- **Relationships:** What value flows between your system and these actors

**Deep Dive: What Counts as "External"?**

- If you can't change its code or redeploy it independently → External
- If it fails and your system must handle that gracefully → External
- Examples: Payment gateways, email providers, third-party APIs, even other teams' microservices

**Staff Challenge Questions:**

- *Have you identified every single external dependency?* Don't forget the "invisible" ones like DNS, NTP, or a shared file system
- *Who are the users?* Do "Admins" and "Customers" access the system through different entry points? Different authentication?
- *What happens if [External System X] is down for 4 hours?* Does your system degrade gracefully or completely fail?

---

### 1.2. Containers (Level 2)

**The Concept:** A "Container" is not just a Docker container. It is **any separately deployable unit** that executes code or stores data.

**What You Document:**

- **Containers:** List each deployable unit (e.g., Web App, API Server, Background Worker, Database, Cache, Queue)
- **Responsibility:** What is each container's job?
- **Protocols:** How do containers talk? (HTTPS, gRPC, AMQP, direct DB connection)

**Deep Dive: What Makes Something "Separate"?**

- **Separately Deployable:** If you change code in Container A and have to redeploy Container B to make it work, they might not actually be separate
- **Data Stores Are Containers:** A PostgreSQL database, Redis cache, and S3 bucket are all "containers" because they're independent deployment units with their own lifecycle
- **Not Just Technology:** A "Mobile App" is a container even though it runs on user devices

**Staff Challenge Questions:**

- *Why did you choose a Single Page App (SPA) over Server-Side Rendering?* What's the trade-off? (SPA = richer interactivity vs SSR = better SEO and initial load)
- *Does the "Worker Service" need to be separate from the "API Service"?* Can they share a process? What do we gain/lose? (Trade-off: Operational complexity vs. resource isolation and independent scaling)
- *Why is the Database a separate container from the API?* (Seems obvious, but forces thinking about: data persistence lifecycle, backup/restore independence, connection pooling, multi-tenancy)

---

### 1.3. Components (Level 3)

**The Concept:** Decompose each container into its internal modules. These are the architectural building blocks *inside* a deployable unit.

**What You Document:**

- **Components:** Logical groupings of functionality (e.g., `UserController`, `OrderService`, `PaymentGateway`, `AuditLogger`)
- **Interfaces:** How do components talk to each other within the container?

**Deep Dive: Components vs. Containers**

- **Container boundary = deployment boundary** (network call, process boundary)
- **Component boundary = code boundary** (function call, module import)
- Components are about code organization and testability, not deployment

**Staff Challenge Questions:**

- *Why is `AuditLogger` a separate component?* Does it have its own data store? Or is this just for separation of concerns in code?
- *If two components share a database table, are they really separate?* Or is this just splitting code for no architectural benefit?
- *Which components have external dependencies?* (Those are your testing pain points and failure surfaces)

---

## Phase 2: Domain & Data Modeling

Bad data models are the **hardest technical debt to fix**. Once you have production data shaped wrong, migration is expensive and risky. We model the business reality first, before thinking about databases.

### 2.1. Entity Catalog

**The Concept:** Entities are the "nouns" of your domain. They represent the core concepts your business cares about.

**What You Document:**

- **Entity Name:** (e.g., User, Order, Product, Invoice)
- **Attributes:** Each field with its type, constraints, optionality
- **Identifying Attributes:** What uniquely identifies this entity? (Often an ID, but sometimes a natural key like email or SKU)
- **Lifecycle States:** How does this entity change over time? (e.g., Order: Draft → Pending → Paid → Shipped → Delivered)

**Deep Dive: IDs vs. Natural Keys**

- **Synthetic ID (UUID, auto-increment):** Never changes, good for databases
- **Natural Key (email, SSN, product SKU):** Meaningful to humans, but can change (people change emails, products get rebranded)
- **The Trap:** Using a natural key as your primary identifier means updates ripple everywhere

**Staff Challenge Questions:**

- *Can this attribute ever be NULL?* Don't just default everything to "optional." Force the question: "Is this data *always* present, or do we have a workflow where it gets filled in later?"
- *What's the lifecycle of this entity?* Does it get soft-deleted or hard-deleted? If soft-deleted, how do you prevent duplicate natural keys?
- *Is this a "Value Object" or an "Entity"?* (Entity = has identity and lifecycle. Value Object = just data with no identity, like an Address or Money amount)

---

### 2.2. Entity Relationships

**The Concept:** Entities don't exist in isolation. Relationships define how they're connected, and **cardinality** defines the scale of that connection.

**What You Document:**

- **Cardinality:** For each relationship, specify:
  - **1:1** (One-to-One): A User has one Profile
  - **1:N** (One-to-Many): A User has many Orders
  - **N:M** (Many-to-Many): Students enroll in many Classes; Classes have many Students
- **Referential Integrity:** Is this relationship enforced by the database (foreign key) or just by application logic?
- **Ownership:** Which entity "owns" the other? (If you delete a User, do their Orders disappear?)
- **Cascade Behavior:** What happens on parent deletion? (CASCADE, SET NULL, RESTRICT)

**Deep Dive: The Terror of "N"**

When you see **1:N**, ask: "How big is N?"

- If N can be **unbounded** (a user could have 1 million orders over their lifetime), you cannot naively fetch "all related records" in one query
- You must design for **pagination** or **archiving** from day one
- Example: GitHub issues. If a repo can have millions of issues, "show all issues" isn't a query you can run

**Staff Challenge Questions:**

- *What happens when we delete a User?* Do we CASCADE delete their Orders (data loss)? Keep them as orphans (broken foreign keys)? Or prevent deletion if Orders exist (RESTRICT)?
- *Which relationships are "strong" vs. "weak"?* Strong = enforced foreign key. Weak = just an ID stored in a field with no database-level enforcement
- *For this N:M relationship, do we need to store metadata on the relationship itself?* (e.g., Student-Class enrollment might need a `grade` field → you need a junction table with attributes)

---

### 2.3. Transactional Boundaries

**The Concept:** Not all operations can be atomic. You must explicitly decide: "What *must* succeed or fail together?"

**What You Document:**

- **Atomic Units:** Which entities must be created/updated/deleted in a single transaction?
- **Consistency Requirements:** Strong (ACID) or Eventual?
- **Isolation Needs:** Can two operations on the same entity run concurrently, or must they be serialized?
- **Compensation Patterns:** If this operation spans systems that can't do distributed transactions, how do we roll back on failure?

**Deep Dive: ACID vs. Eventual Consistency**

- **ACID (Strong Consistency):** The database guarantees that all parts of a transaction commit or none do. Required for: inventory, financial balances, anything where "partial success" is unacceptable
- **Eventual Consistency:** You accept that different parts of the system might be temporarily out of sync, but will "eventually" converge. Acceptable for: social feeds, search indexes, analytics

**The Trap:** Defaulting to ACID for everything. Strong consistency is expensive at scale. If you can tolerate 5 seconds of staleness, you unlock caching, read replicas, and async processing.

**Staff Challenge Questions:**

- *You marked "Create User + Profile" as atomic. Why?* What breaks if the User exists but the Profile doesn't? (Answer: Probably nothing *breaks*, but you might show broken UI. Is that worth the transaction overhead?)
- *What happens if step 2 of a 3-step process fails?* Do we have a rollback mechanism? Or do we retry? Or do we compensate (undo step 1)?
- *Can we make this eventually consistent instead?* (Often the answer is yes, and it buys you massive scalability)

---

## Phase 3: Access Pattern Definition

**This is the most critical section.** We design storage based on **how we read and write**, not just how data "looks."

Junior engineers model entities and then figure out queries later. Staff engineers enumerate queries *first*, because that's what dictates indexes, denormalization, and database technology choice.

---

### 3.1. Read Access Pattern Inventory

**The Concept:** Every way your system reads data must be documented and analyzed.

**What You Document (Per Pattern):**

```
AP-XXX: [Pattern Name]
  - Access Type: [Point lookup | One-to-many navigation | Global filter | Multi-criteria filter | Aggregation]
  - Lookup Key: [What identifies the data being retrieved]
  - Sort Order: [If applicable]
  - Frequency: [Requests per second]
  - Latency SLA: [p50/p95/p99 targets]
  - Consistency Requirement: [Strong | Eventual | Read-your-writes]
  - Result Cardinality: [0-1 record | 0-N bounded | 0-N unbounded]
  - Pagination: [Required | Not needed]
  - Data Coverage: [Selective (indexed) | Full scan required]
```

**Deep Dive: Consistency Models**

- **Strong Consistency:** If I write X, then immediately read X, I *must* see the new value
  - Required for: Inventory counts (can't oversell), bank balances, seat reservations
- **Read-Your-Writes:** *I* must see my own writes immediately, but others can see them eventually
  - Good for: Social media (you see your own post instantly, others see it in ~1 second)
- **Eventual Consistency:** If I write X and read X, I *might* see the old value for a few seconds
  - Acceptable for: Feeds, search results, analytics dashboards

**The Trap:** Marking everything "Strong Consistency" because it's "safer." Strong consistency kills performance and prevents caching.

**Deep Dive: Result Cardinality (The "N" Problem Again)**

- **0-1 record:** Safe. A single row lookup.
- **0-N bounded:** "A user's last 100 orders" — you can fetch all in one query
- **0-N unbounded:** "All of a user's orders over 10 years" — you MUST paginate or you'll run out of memory

**Staff Challenge Questions:**

- *You listed "Strong Consistency" for the User Feed (AP-XXX). Why?* Does it break the business if a user doesn't see their own post for 2 seconds? (Hint: Probably not. Switch to "Read-Your-Writes" and unlock caching)
- *For AP-005 (Search Products), you're filtering by category AND price range AND rating. Do we need a multi-column index?* Or is this a job for a search engine (Elasticsearch)?
- *This pattern is marked "Full Scan." How many rows are we scanning?* If it's millions, this won't scale. Do we need a batch job instead?
- *What's the p99 latency target?* Don't say "<100ms" if you haven't measured whether that's achievable with your data size

**Example:**
```
AP-001: Retrieve single User by unique identifier
  - Access Type: Point lookup
  - Lookup Key: userId (unique)
  - Frequency: 10k req/sec
  - Latency SLA: <50ms p99
  - Consistency Requirement: Read-your-writes (strong)
  - Result Cardinality: 0 or 1 record

AP-002: Retrieve Orders belonging to a User, ordered by creation time
  - Access Type: One-to-many navigation
  - Lookup Key: userId (foreign reference)
  - Sort Order: createdAt DESC
  - Frequency: 500 req/sec
  - Latency SLA: <100ms p99
  - Consistency Requirement: Eventual (5-second staleness acceptable)
  - Result Cardinality: 0 to N records
  - Pagination: Required (page size: 20)
  - Total Result Size: Unbounded (grows with user activity)
```

---

### 3.2. Write Pattern Inventory

**The Concept:** Every way your system mutates state must be documented, because writes are where things go wrong (concurrency, data loss, race conditions).

**What You Document (Per Pattern):**

```
WP-XXX: [Pattern Name]
  - Operation Type: [INSERT | UPDATE | DELETE]
  - Affected Entities: [List of entities and count]
  - Volume: [Writes per second, average and peak]
  - Atomicity Requirement: [Single entity | Multi-entity | Distributed]
  - Uniqueness Constraints: [Fields that must be globally unique]
  - Conditional Logic: [Preconditions for the write to succeed]
  - Concurrency Pattern: [Last-write-wins | Optimistic locking | Pessimistic locking]
  - Idempotency: [Required | Not required]
  - Durability: [Immediate (synchronous) | Async acceptable]
  - Side Effects: [Events triggered, cascades, external system calls]
```

**Deep Dive: Concurrency Control**

What happens when **two users edit the same record at the exact same time**?

- **Last-Write-Wins:** Whoever saves last overwrites the previous work. Simple, but dangerous for collaborative editing (one person's changes vanish)
- **Optimistic Locking:** The system checks a `version` number. If the version changed since you read it, the write fails with "409 Conflict" and the client must refresh and retry
- **Pessimistic Locking:** The first user to read the record *locks* it. Other users can't even read until the lock is released. Rare in web apps (too slow)

**Deep Dive: Idempotency**

**Definition:** Performing an operation multiple times has the same effect as performing it once.

**Why it matters:** Networks are unreliable. Clients retry requests. If `CreateOrder` isn't idempotent, retrying creates duplicate orders.

**How to implement:**
- Client sends a unique `Idempotency-Key` header (a UUID they generate)
- Server checks: "Have I seen this key before?"
  - If yes → Return the previous response (don't redo the work)
  - If no → Process the request and store the key + response

**Staff Challenge Questions:**

- *WP-002: Two admins try to approve the same invoice at the same time. How do we prevent double-payment?* (Answer: Optimistic locking on the invoice's `version` field)
- *Is this write idempotent?* Walk me through what happens if the client retries 3 times due to network timeout
- *You marked "Immediate Durability." Why?* Can this write be async (queued for later)? What's the business impact if we lose this write due to a crash before it's persisted?
- *This writes to 3 entities. Are they in a transaction?* What happens if entity 2 fails to write? (Partial success is almost always a bug)

**Example:**
```
WP-001: Create new User account
  - Operation Type: INSERT
  - Affected Entities: User (1) + UserProfile (1)
  - Volume: 100 writes/sec average, 500/sec peak
  - Atomicity Requirement: Both entities must be created together
  - Uniqueness Constraints: email must be globally unique
  - Idempotency: Required (must be retry-safe)
  - Durability: Immediate (must be persisted before acknowledgment)
  - Side Effects: Triggers UserCreated event (asynchronous)

WP-002: Update Order status
  - Operation Type: UPDATE (conditional)
  - Affected Entities: Order (1)
  - Volume: 2k writes/sec
  - Atomicity Requirement: Single entity
  - Conditional Logic: Only allow if currentStatus → newStatus is valid transition
  - Concurrency Pattern: Optimistic locking (version-based)
  - Durability: Immediate
  - Side Effects: Triggers OrderStatusChanged event
```

---

### 3.3. Co-Access & Locality Patterns

**The Concept:** Some entities are **always accessed together**. Identifying these patterns is the key to deciding whether to denormalize, embed documents, or pre-join data.

**What You Document:**

```
Locality Pattern X: [Pattern Name]
  - Entities: [List of entities always accessed together]
  - Access Pattern Reference: [Which AP-XXX requires this]
  - Read Frequency: [How often is this co-access performed]
  - Update Frequency: [How often do any of these entities change]
  - Design Implication: [What this suggests about storage]
  - Trade-off: [Cost of co-location vs. separation]
```

**Deep Dive: When to Denormalize**

**Denormalization = duplicating data to avoid joins.**

Example: Storing `customerName` in the `orders` table instead of joining to `customers`.

**When it makes sense:**
- High read frequency (thousands/sec)
- Low update frequency (customer names rarely change)
- Read latency is critical

**When it's dangerous:**
- Data changes frequently (now you have write amplification: update in 2 places)
- Data gets out of sync (what if customer name changes in `customers` but not `orders`?)

**Staff Challenge Questions:**

- *You identified "Order + LineItems" as always accessed together. Should we embed LineItems in the Order document?* What if an order has 1000 line items? Does that break your document size limit?
- *For "User Activity Timeline" (Posts + Comments + Likes), you're doing a 3-table join. How often is this query run?* If it's thousands/sec, maybe we pre-compute and cache it. If it's 10/sec, the join is fine
- *What's the trade-off of denormalizing customer email into orders?* (Pro: Fast reads. Con: If customer changes email, we have stale data in old orders — is that okay?)

**Example:**
```
Locality Pattern 1: Order Aggregate
  - Entities: Order + LineItems + Customer (subset of fields)
  - Access Pattern: AP-004 (always retrieved together)
  - Read Frequency: High (2k req/sec)
  - Update Frequency: Low (Order: write-once, LineItems: immutable after creation)
  - Design Implication: Strong candidate for storage co-location
  - Trade-off: Write amplification if Customer data is duplicated vs. join cost
```

---

### 3.4. Data Derivation & Aggregation Patterns

**The Concept:** Some data is **computed from other data** (e.g., "total order count" is derived by counting rows). You must decide: compute on-read, compute on-write, or batch-compute?

**What You Document:**

```
Derived Data X: [Name]
  - Source: [How is it computed from base entities]
  - Update Trigger: [Which write patterns affect this derived data]
  - Consistency Model: [Strong | Eventual]
  - Staleness Impact: [What breaks if this is stale for 1 minute? 1 hour?]
  - Maintenance Options:
    - Option A: Computed on-demand (query-time aggregation)
    - Option B: Maintained incrementally (updated on every write)
    - Option C: Batch recomputation (hourly/daily job)
  - Decision Criteria: [Read frequency vs write frequency vs staleness tolerance]
```

**Deep Dive: The Three Strategies**

| Strategy | When to Use | Trade-off |
|----------|------------|-----------|
| **On-Demand (Query-time)** | Low read frequency, simple aggregation | Slow queries, high DB load |
| **Incremental (Write-time)** | High read frequency, low write frequency | Write amplification, complexity |
| **Batch (Scheduled job)** | Staleness acceptable (hours/days), complex computation | Not real-time, job failures |

**Example:** `User.orderCount`
- **On-demand:** `SELECT COUNT(*) FROM orders WHERE userId = X` (slow if user has 10k orders)
- **Incremental:** Maintain a counter, increment on `PlaceOrder`, decrement on `CancelOrder` (fast reads, but buggy if counter drifts out of sync)
- **Batch:** Nightly job recomputes all counts (acceptable if this is just for analytics)

**Staff Challenge Questions:**

- *You're computing "total revenue by user" on every page load. How long does that query take if a user has 50,000 orders?* (Hint: Too long. Pre-compute it.)
- *If we maintain this derived data incrementally, what happens if the update fails?* Do we retry? Do we have a reconciliation job to fix drift?
- *Is this derived data mission-critical or nice-to-have?* If it's for an executive dashboard that's viewed once a day, batch recomputation is fine

**Example:**
```
Derived Data 1: User.orderCount
  - Source: COUNT(Orders) WHERE userId = X
  - Update Trigger: WP-003 (Place Order), order deletion
  - Consistency Model: Eventually consistent (acceptable lag: 1 minute)
  - Staleness Impact: Low (used for analytics, not critical path)
  - Maintenance Strategy: 
    - Option A: Computed on-demand (query-time aggregation)
    - Option B: Maintained counter (updated on write)
    - Option C: Batch recomputation (daily/hourly)
  - Selection Criteria: Read frequency vs. write frequency, staleness tolerance
```

---

### 3.5. Read/Write Characteristics Matrix

**The Concept:** Summarize all patterns in a table so you can **spot optimization opportunities** at a glance.

| Access Pattern | Frequency | Read:Write Ratio | Latency SLA | Consistency | Data Volume | Selectivity |
|---------------|-----------|------------------|-------------|-------------|-------------|-------------|
| AP-001 | 10k/s | 100:1 | <50ms p99 | Strong | 1 record | Point |
| AP-002 | 500/s | 50:1 | <100ms p99 | Eventual | 0-100 records | Selective |
| AP-003 | 10/s | 1:1 | <500ms p99 | Eventual | 10k records | Full scan |

**What This Tells You:**

- **AP-001:** High frequency + point lookup → **Must be indexed, cache heavily**
- **AP-002:** Medium frequency + selective → **Index required, cache for 30s**
- **AP-003:** Low frequency + full scan → **Don't optimize the database; use a batch job or OLAP store**

**Staff Challenge Questions:**

- *AP-003 is a full-table scan. How many rows are in this table today? In 1 year?* If it grows to 10 million rows, this query will time out
- *AP-001 has a 100:1 read:write ratio. Why aren't we caching this?* (If it's "strong consistency," challenge that requirement)

---

## Phase 4: Interface Definition (The Contract)

We separate the **"What"** (Interface) from the **"How"** (Implementation). Ideally, we use **CQRS** (Command Query Responsibility Segregation): Commands change state, Queries read state, Events notify about changes.

---

### 4.1. Commands (Write Side)

**The Concept:** A Command is an **imperative instruction to change the system's state**. It represents intent: "I want to register a user" or "I want to cancel this order."

**What You Document:**

```
Command: [VerbNoun]
  - Maps to: [Write Pattern WP-XXX from Phase 3.2]
  - Payload: [Required data structure — only what's needed to execute]
  - Idempotency: [Mechanism: header, embedded key, etc.]
  - Validation Rules: [Business rules checked before execution]
  - Success Response: [HTTP status + minimal payload]
  - Error Responses: [All possible failure modes with status codes]
  - Events Emitted: [Which events fire on success]
  - Transactional Guarantee: [What commits atomically]
```

**Naming Convention:** `VerbNoun` (e.g., `RegisterUser`, `ApproveInvoice`, `CancelOrder`)

**Deep Dive: Idempotency (Again, Because It's Critical)**

**Bad Implementation:**
```
POST /orders
{ "productId": "abc", "quantity": 2 }
```
If the client retries this 3 times due to timeout, you create 3 orders. 🐛

**Good Implementation:**
```
POST /orders
Headers: { "Idempotency-Key": "client-generated-uuid" }
{ "productId": "abc", "quantity": 2 }
```
Server logic:
1. Check if `Idempotency-Key` exists in cache/DB
2. If yes → return the stored response (don't create a new order)
3. If no → process the command, store the key + response, return success

**Deep Dive: Validation vs. Preconditions**

- **Validation:** Is the *payload* well-formed? (e.g., email format, positive price)
- **Preconditions:** Is the *system state* correct for this command? (e.g., "Can't cancel an order that's already shipped")

**Staff Challenge Questions:**

- *Walk me through a retry scenario.* Client calls `PlaceOrder` with idempotency key `abc-123`. Network times out. Client retries with the same key. What does the server do? (Answer: Check the key, see it already processed, return the stored response)
- *You return the full `Order` object in the success response. Why?* Commands should return **minimal data** (just the ID), because the client can query for the full object if needed. Returning everything couples your write and read models
- *What if two validation rules conflict?* (e.g., "price must be positive" but also "price must match product catalog"). Which error do we return first?
- *This command writes to 3 entities. If entity 2 fails, do we rollback entities 1 and 3?* (If not, you have data corruption)

**Example:**
```
Command: RegisterUser
  - Maps to: WP-001
  - Payload: 
    {
      "email": "string (email format, required)",
      "password": "string (min 8 chars, required)",
      "profile": {
        "firstName": "string (required)",
        "lastName": "string (required)"
      }
    }
  - Idempotency: Via X-Idempotency-Key header (client-generated UUID)
  - Validation Rules:
    - Email must not already exist (check before write)
    - Password must meet complexity requirements
  - Success Response: 201 Created
    {
      "userId": "uuid",
      "createdAt": "iso8601"
    }
  - Error Responses:
    - 409 Conflict: Email already registered
    - 400 Bad Request: Validation failures
    - 500 Internal Server Error: System failure
  - Events Emitted: UserRegistered
  - Transactional Guarantee: Strong (both User and UserProfile created or neither)
```

---

### 4.2. Queries (Read Side)

**The Concept:** A Query **retrieves data without changing state**. It's a question: "What is user X's profile?" or "What are the pending invoices?"

**What You Document:**

```
Query: [NounBased or Question]
  - Maps to: [Access Pattern AP-XXX from Phase 3.1]
  - Parameters: [Path params, query params, headers]
  - Caching Strategy: [TTL, invalidation triggers, cache key format]
  - Response: [HTTP status + DTO structure]
  - Error Responses: [404, 403, etc.]
  - Consistency: [What consistency guarantee does this provide]
```

**Naming Convention:** Noun-based or question-style (e.g., `GetUserProfile`, `ListPendingInvoices`, `SearchProducts`)

**Deep Dive: DTOs (Data Transfer Objects)**

**The Rule:** The JSON you return is **not your database schema**. It's a contract with the client.

**Bad (Database Leakage):**
```json
{
  "user_id": 123,
  "password_hash": "$2b$10$...",
  "internal_flag": true,
  "created_at": "2024-01-01T00:00:00Z"
}
```
You just leaked sensitive fields and exposed internal column naming.

**Good (DTO):**
```json
{
  "userId": "123",
  "email": "user@example.com",
  "profile": {
    "firstName": "Jane",
    "lastName": "Doe"
  },
  "joinedAt": "2024-01-01"
}
```
Clean, safe, client-friendly. If you rename the DB column `created_at` to `registration_timestamp`, the API contract doesn't break.

**Deep Dive: Pagination (Cursor vs. Offset)**

**Offset-based pagination:**
```
GET /orders?limit=20&offset=40
```
- **Pro:** Simple. "Give me page 3" is easy to understand
- **Con:** Unstable. If someone inserts a row while you're paginating, you might see duplicates or skip rows

**Cursor-based pagination:**
```
GET /orders?limit=20&cursor=eyJsYXN0SWQiOjEyMywibGFzdFRpbWVzdGFtcCI6IjIwMjQtMDEtMDEifQ==
```
- **Pro:** Stable. The cursor encodes "where you left off" (e.g., last ID + timestamp), so inserts don't break pagination
- **Con:** Can't jump to "page 5" directly. Must paginate sequentially

**Staff Challenge Questions:**

- *You're caching this query for 5 minutes. What happens if the underlying data changes in that time?* (Answer: Clients see stale data. Is that acceptable for this use case?)
- *This query returns a User object with 50 fields. Does the mobile app need all 50?* (Over-fetching wastes bandwidth. Consider GraphQL or field filtering)
- *How do you invalidate the cache when a write happens?* (e.g., On `UpdateUser` command, delete cache key `user:profile:{userId}`)
- *What if pagination cursor is tampered with?* (It's base64-encoded, not encrypted. Clients can decode and modify it. Do you validate it?)

**Example:**
```
Query: GetUserProfile
  - Maps to: AP-001
  - Parameters: userId (path parameter, UUID)
  - Caching Strategy: 
    - TTL: 5 minutes
    - Invalidation: On any user update command
    - Cache key: "user:profile:{userId}"
  - Response: 200 OK
    {
      "userId": "uuid",
      "email": "string",
      "profile": {
        "firstName": "string",
        "lastName": "string"
      },
      "createdAt": "iso8601",
      "updatedAt": "iso8601"
    }
  - Error Responses: 
    - 404 Not Found: User does not exist
    - 403 Forbidden: Not authorized to view profile
  - Consistency: Read-your-writes for the creating user, eventual for others

Query: ListUserOrders
  - Maps to: AP-002
  - Parameters:
    - userId (path parameter, UUID)
    - cursor (query param, opaque token, optional)
    - limit (query param, integer, default 20, max 100)
  - Caching Strategy: Short TTL (30 seconds) due to high update frequency
  - Response: 200 OK
    {
      "orders": [
        {
          "orderId": "uuid",
          "status": "string",
          "totalAmount": "decimal",
          "itemCount": "integer",
          "createdAt": "iso8601"
        }
      ],
      "nextCursor": "string | null",
      "hasMore": "boolean"
    }
  - Error Responses:
    - 404 Not Found: User does not exist
    - 400 Bad Request: Invalid cursor or limit
  - Pagination: Cursor-based (stable across concurrent modifications)
  - Sorting: createdAt DESC (implied, not parameterized)
  - Consistency: Eventual (5-second staleness acceptable)
```

---

### 4.3. Events (Side Effects)

**The Concept:** An Event is a **fact about something that happened in the past**. It's notification-based: "UserRegistered" or "OrderPlaced." Other parts of the system listen to events and react.

**What You Document:**

```
Event: [NounPastTenseVerb]
  - Triggered By: [Which Command(s) from 4.1 cause this]
  - Payload: [Immutable data describing what happened]
  - Delivery Guarantee: [At-most-once | At-least-once | Exactly-once]
  - Ordering Guarantee: [None | Per-entity | Global]
  - Partition Key: [For ordered delivery in distributed systems]
  - Consumers: [Which services/components react to this event]
```

**Naming Convention:** `NounPastTenseVerb` (e.g., `UserRegistered`, `InvoiceApproved`, `OrderShipped`)

**Deep Dive: Side Effects & Decoupling**

**The Problem:** If you put side effects (sending emails, updating analytics) **inside the main transaction**, failures in side effects kill the main operation.

**Example (Bad — Coupled):**
```
1. Insert User into database
2. Send welcome email ← Email server is down
3. Transaction fails, user not created 😞
```

**Example (Good — Decoupled via Events):**
```
1. Insert User into database
2. Publish "UserRegistered" event
3. Transaction succeeds ✅
4. Email service (separate process) consumes the event and sends email
   - If email fails, it retries later
   - Main transaction is unaffected
```

**Deep Dive: Delivery Guarantees**

- **At-most-once:** Fire and forget. If the message is lost, too bad. (Fast, but you lose data)
- **At-least-once:** Retry until acknowledged. Message might be delivered multiple times. (Reliable, but consumers must be idempotent)
- **Exactly-once:** Delivered exactly once, guaranteed. (Theoretically ideal, practically very expensive and complex)

**The Trap:** Trying to achieve exactly-once delivery. It's almost impossible in distributed systems. Instead, design **idempotent consumers** (so duplicate messages are harmless).

**Staff Challenge Questions:**

- *What happens if the database commit succeeds, but publishing the event fails?* (You have a committed transaction but no event. Other systems don't know it happened. How do you recover?)
- *This event payload contains the full User object with 50 fields. Why?* **Fat events** (contain all data) mean consumers don't need to query. **Thin events** (just IDs) mean consumers must query for details. Which is better depends on your architecture
- *If the email service is down for 2 hours, does it replay all events when it comes back up?* (Depends on retention policy and consumer offset management)
- *How do you prevent duplicate event processing?* (Consumers must track "have I processed event ID `xyz` before?" — use a deduplication table or idempotency key)

**Example:**
```
Event: UserRegistered
  - Triggered By: RegisterUser command (WP-001)
  - Payload: (Immutable)
    {
      "eventId": "uuid",
      "occurredAt": "iso8601",
      "userId": "uuid",
      "email": "string",
      "profileCreated": "boolean"
    }
  - Delivery Guarantee: At-least-once
  - Ordering Guarantee: Per-user ordering (userId as partition key)
  - Partition Key: userId
  - Consumers:
    - Email Service: Send welcome email
    - Analytics Service: Track user acquisition
    - Audit Log: Compliance recording

Event: OrderPlaced
  - Triggered By: PlaceOrder command (WP-003)
  - Payload: (Immutable)
    {
      "eventId": "uuid",
      "occurredAt": "iso8601",
      "orderId": "uuid",
      "userId": "uuid",
      "totalAmount": "decimal",
      "itemCount": "integer",
      "lineItems": [
        {
          "productId": "uuid",
          "quantity": "integer"
        }
      ]
    }
  - Delivery Guarantee: At-least-once
  - Ordering Guarantee: Per-user ordering
  - Partition Key: userId
  - Consumers:
    - Inventory Service: Reserve stock
    - Fulfillment Service: Initiate picking
    - Notification Service: Notify user of order confirmation
    - Analytics Service: Track order metrics
```

---

## Phase 5: Failure Modes & Hard Limits

Junior engineers design for the **Happy Path**. Staff engineers design for **failure**.

### 5.1. Hard Limits

**The Concept:** Every system has physical constraints. Document them explicitly so you know when you'll hit a wall.

**What You Document:**

- **Max Payload Size:** (e.g., 5MB for HTTP body, 256KB for message queue)
- **Rate Limits:** (e.g., 100 requests/min per user, 10k requests/sec total)
- **Timeouts:** (e.g., Database queries abort after 5s, HTTP requests timeout after 30s)
- **Connection Limits:** (e.g., DB supports 100 concurrent connections, Redis pool size 50)
- **Storage Limits:** (e.g., Max 10k records per query, Max 1GB per file upload)

**Staff Challenge Questions:**

- *If this service receives 10x the expected traffic, what breaks first?* The DB CPU? Memory? Network bandwidth?
- *What's the largest payload a client could send?* If someone uploads a 500MB file, does it crash your server?
- *How many concurrent users can you handle before connection pooling fails?*

---

### 5.2. Failure Scenarios

**The Concept:** Enumerate specific failure modes and your system's response.

**What You Document:**

```
Scenario: [What fails]
  - Impact: [What breaks in your system]
  - Detection: [How do you know it's happening]
  - Behavior: [What does the system do]
  - Recovery: [How do you fix it]
```

**Examples:**

```
Scenario: Database is down
  - Impact: All reads and writes fail
  - Detection: Health check fails, connection pool exhausted
  - Behavior: API returns 503 Service Unavailable
  - Recovery: Auto-retry with exponential backoff, alert on-call engineer

Scenario: Payment gateway (Stripe) is slow (>5s response time)
  - Impact: Checkout requests hang
  - Detection: Timeout after 2s
  - Behavior: Return 202 Accepted, queue payment for async processing
  - Recovery: Background worker retries payment, notifies user via email

Scenario: Message queue is full
  - Impact: Cannot publish events
  - Detection: Publish returns error "Queue full"
  - Behavior: Drop event and log error (acceptable if events are non-critical), OR reject the command (if events are critical)
  - Recovery: Scale up queue, or add backpressure (HTTP 429 Too Many Requests)

Scenario: "Poison message" crashes consumer
  - Impact: Consumer repeatedly fails and restarts
  - Detection: Consumer lag grows, error logs repeat
  - Behavior: After N retries, move message to dead-letter queue
  - Recovery: Manual inspection, fix bug, replay from dead-letter queue
```

**Staff Challenge Questions:**

- *What's your "blast radius" if the database crashes?* (Does it take down 1 service or 10?)
- *How do you recover from a "poison message"?* (A message that crashes the consumer every time it's processed — do you have a dead-letter queue?)
- *If a third-party API is down, do you fail fast or retry?* (Fail fast = don't waste resources. Retry = might succeed on transient errors. Depends on criticality)
- *What happens if your cache goes down?* (Thundering herd problem: all requests hit the database at once)

---

## Summary: How to Use This Document

### In Design Reviews

1. **Phase 1-2:** Sketch on a whiteboard. Get team alignment on boundaries and entities
2. **Phase 3:** The hard part. Force the team to enumerate **every single access pattern**. This is where you catch missing indexes and performance cliffs
3. **Phase 4:** Formalize the API contract. Review for idempotency, error handling, and payload size
4. **Phase 5:** Play "What if?" Simulate failures. If you can't answer "What happens when X fails," the design isn't done

### As a Teaching Tool

- **For Junior Engineers:** Walk through the "Deep Dive" sections together. Explain *why* consistency matters, *why* pagination is hard, *why* idempotency is non-negotiable
- **For Mid-Level Engineers:** Have them fill out the template for their feature. Review their answers. Ask the "Staff Challenge Questions"
- **For Staff Engineers:** Use this as a checklist. If you can't answer every question, dig deeper

### The Goal

By the end of this process, you should be able to answer:

✅ What are all the external dependencies?  
✅ What entities exist and how are they related?  
✅ How will data be read and written?  
✅ What are the SLAs and consistency requirements?  
✅ What's the public API contract?  
✅ What happens when things fail?

**If you can't answer these questions, you're not ready to code.**

---

## Appendix: Quick Reference Templates

### Entity Template
```
Entity: [Name]
  - Attributes:
    - [name]: [type] [constraints] [optional/required]
  - Identifying Attribute: [primary identifier]
  - Lifecycle: [state1] → [state2] → [state3]
  - Relationships:
    - [1:1|1:N|N:M] with [OtherEntity]
```

### Access Pattern Template
```
AP-XXX: [Pattern Name]
  - Access Type: [type]
  - Lookup Key: [key]
  - Frequency: [req/sec]
  - Latency SLA: [ms]
  - Consistency: [Strong|Eventual|Read-your-writes]
  - Result Cardinality: [0-1|0-N bounded|0-N unbounded]
  - Pagination: [Yes|No]
```

### Write Pattern Template
```
WP-XXX: [Pattern Name]
  - Operation Type: [INSERT|UPDATE|DELETE]
  - Affected Entities: [list]
  - Volume: [writes/sec avg], [writes/sec peak]
  - Atomicity: [Single|Multi-entity|Distributed]
  - Idempotency: [Required|Not required]
  - Side Effects: [events, cascades]
```

### Command Template
```
Command: [VerbNoun]
  - Maps to: WP-XXX
  - Payload: { [structure] }
  - Idempotency: [mechanism]
  - Success: [status code] { [minimal response] }
  - Errors: [status codes and meanings]
  - Events: [EventName]
```

### Query Template
```
Query: [NounBased]
  - Maps to: AP-XXX
  - Parameters: [list]
  - Caching: [TTL, strategy]
  - Response: [status] { [DTO structure] }
  - Errors: [status codes]
```

### Event Template
```
Event: [NounPastTenseVerb]
  - Triggered By: [Command]
  - Payload: { [immutable data] }
  - Delivery: [At-most|At-least|Exactly]-once
  - Ordering: [None|Per-entity|Global]
  - Consumers: [list of services]
```

### Failure Scenario Template
```
Scenario: [What fails]
  - Impact: [consequences]
  - Detection: [how you know]
  - Behavior: [system response]
  - Recovery: [remediation steps]
```
