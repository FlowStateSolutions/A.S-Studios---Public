# A.S. Studios: System Architecture Case Study

**Project Status:** Production Architecture Specifications (Public Spec) | Live application is in active development (Private Repo).

> ### 🔒 Code Privacy & Intellectual Property Notice
> The actual codebase, database migrations, and live travel data belong to Alexa Safaris and sit securely inside a private repository. This public repository serves as an open system design case study to showcase the relational database modeling, architectural design patterns, and state-management strategies used to build the platform.

---

## 🏗️ Technical Architecture & Stack

* **Frontend Engine:** React, TypeScript, Tailwind CSS
* **Backend & Database (BaaS):** Supabase (PostgreSQL relational schema modeling, database views, atomic transaction functions, and bulk upsert operations)
* **Document Generation:** Client-facing document layout engine (Custom exports configured to scrub proprietary cost vectors programmatically)
* **AI-Assisted Architecture:**
  * Accelerated MVP development using Lovable for rapid full-stack prototyping.
  * Leveraged Gemini to migrate and translate legacy TSQL patterns into optimized PostgreSQL.
  * Utilized AI as a dedicated systems-design partner to stress-test architectural roadmaps and database normalization rules.

---

## 📋 The Real-World Problem

In luxury safari and travel planning, operators manage highly volatile, multi-leg itineraries. The planning process involves a messy, non-linear feedback loop with clients. Operators need to evaluate various suppliers, track internal cost variations, swap legs out, and re-order trips on the fly. 

The core challenge isn't automation—it is **malleability**. 

Building a system with rigid data validation breaks the operator's real-world workflow. **A.S. Studios** is architected to act as an organized digital scratchpad. It gives the operator absolute freedom to input partial data, manually experiment with numbers, and iteratively cull trip legs, while acting as a secure engine that generates clean, professional, and consistent itinerary documents for the end client.

---

## 🔄 The Phase 1 Operational Pipeline

The system splits the lifecycle of a trip itinerary into three distinct layers: a static data foundation, a highly fluid operational sandbox, and a locked archival snapshot.

```text
[ 0. STATIC DATA MANAGEMENT ]
System provides full administrative CRUD access to the unified master catalog, 
ensuring the operator controls the baseline data for all locations and activities.
                         |
                         v
[ 1. THE WORKSHOP SCRATCHPAD ]
User drafts trip legs in `wo_itinerary_items` using a high-performance search engine.
  - Relational foreign keys connect to the master suppliers catalog.
  - Fully malleable cost data fields (can be NULL or populated at will).
  - Seamless drag-and-drop reordering optimized via Bulk Upsert.
                         |
                         v
[ 2. ITERATIVE CLIENT FEEDBACK ]
User generates cost-free Proposal Documents.
  - Operator adds, removes, or resequences legs based on client feedback.
  - Documents are regenerated programmatically without altering baseline data.
                         |
                         v (User clicks "Finalize Itinerary")
[ 3. ATOMIC SNAPSHOT TRANSACTION ]
Database executes an all-or-nothing RPC function:
  1. Wipes previous finalized data for current Trip ID.
  2. Copies current workshop state into the archival layer.
                         |
                         v
[ 4. FLATTENED ARCHIVAL STORAGE ]
Data is frozen inside `finalized_itinerary_items`.
  - References are flattened into raw, independent text strings.
  - Historical itineraries are completely isolated from future dictionary modifications or deletes.
```

---

🗄️## Database Architecture & Relational Strategy
To maximize query performance and eliminate data fragmentation, the backend utilizes a centralized physical layer paired with logical database abstractions.

1. Unified Supplier Directory (master_suppliers)
Instead of fragmenting locations, transport providers, and activities into separate, disconnected tables, the system normalizes them into a single core table.

Shared Class Attributes: Fields like name, contact_number, description, image_url, and tags are treated as global supplier properties.

Geographic & Value Anchoring: Every supplier type natively inherits country_id, neighborhood_id, and a unified price tier enum (value_tier: Low, Mid, High). This enables high-performance frontend filtering across all supplier types simultaneously.

Frontend UI Abstraction: To maintain a tailored administrative experience, the frontend UI is designed to split each supplier category into its own dedicated management screen. This allows the operator to experience distinct, logical dashboards while the underlying database remains perfectly unified.

2. The Sandbox Layer (wo_itinerary_items)
Tracks the active draft legs of a trip. It maintains strict foreign key constraints back to the master_suppliers table and contains unvalidated numeric/text cost vectors. Rows here are designed to be heavily edited, reordered, or deleted during client negotiations.

3. The Archival Layer (finalized_itinerary_items)
When an itinerary is confirmed, this table records the historical snapshot. Crucially, this table flattens relational data into raw text strings (e.g., writing the actual supplier string value rather than preserving the foreign key ID). This guarantees that if a master supplier record is modified or deleted in the future, the historical client document remains completely unaffected.

---

⚡## Engineering Highlights
🚄 1. Network-Optimized Drag-and-Drop Sorting (Bulk Upsert)
Managing user-driven sequential order (drag-and-drop) over a web connection can quickly become a database bottleneck. If a user moves an item across a large timeline, firing individual UPDATE commands for every shifted row creates immense network overhead and risks race conditions.

The system circumvents this by executing a Bulk Upsert. The frontend calculates the newly arranged sequence in a single state array and transmits one JSON payload to Supabase. The database opens a single transaction, updates the shifted sequences in memory, and returns a single network response, ensuring zero lag and perfect sequential integrity.

🔒 2. All-or-Nothing Finalization (Atomic Transactions)
When an itinerary is finalized, the system replaces the old snapshot with a fresh one using a Delete-All-Then-Insert script mapped to the specific trip_id.

To prevent a network drop or unexpected crash from erasing an itinerary mid-transfer, these operations are bound within a native PostgreSQL Database Transaction. If the subsequent inserts fail, the initial deletion rolls back entirely, ensuring the system never compromises or drops existing user data.

🔍 3. Unified Entity Search Engine
Allowing a user to intuitively search through thousands of unique suppliers requires more than a standard text query. Because the architecture merges all entities into a single master_suppliers table, the frontend search engine can execute highly complex, multi-parameter queries in a single database round-trip. The user can dynamically cross-filter by supplier_type, country_id, and embedded string matching without relying on slow, multi-table JOIN statements, ensuring the trip-building experience remains lightning-fast regardless of database size.

---

🚀 Future Enhancements (Phase 2 Roadmap)
Application-Side Calculation Engine: Introducing structured currency validation (ZAR/USD tracking) and automated margin calculations.

Automated Proposal of Costs (PoC): Generating secondary financial client-facing summaries directly from populated database fields.
