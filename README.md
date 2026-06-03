# A.S. Studios: System Architecture Case Study

**Status:** Application is in active development (Private Repo) | This public page holds both the completed and planned system specs.

> ### 🔒 Heads Up on Code Privacy
> The actual codebase, database schemas, and live travel data belong to Alexa Safaris and sit safely inside a private repository. I built this public page to serve as an open system design case study so you can see how I approached the architecture and database modeling for the project.

---

## 🏗️ Technical Architecture & Stack

* **Frontend Engine:** React, TypeScript, Tailwind CSS
* **Backend & Database (BaaS):** Supabase (PostgreSQL relational schema modeling, foreign key cascade constraints, custom type enumerations, and atomic upsert operations)
* **File Processing Architecture:** `exceljs` / Browser-side local buffer streams (Utilized to inject active, working Excel string formulas directly into downloaded workbooks programmatically)
---

## 📋 Executive Project Summary

In the luxury travel and safari planning industry, travel designers rely heavily on the flexibility, calculation speed, and visual formatting control of spreadsheets. However, managing enterprise operations out of isolated, local files introduces severe data-integrity issues, including naming fragmentation, lack of historical version tracking, and a high risk of data loss.

**A.S. Studios** bridges this gap. Instead of engineering a rigid web-form UI that disrupts the core planning workflow, this application acts as a centralized data aggregator and file-orchestration engine. It couples a permanent, normalized destination knowledge base with a bidirectional pipeline that programmatically streams formula-live spreadsheets and ingests local workbook adjustments back into a secure relational database.

---

## 🗄️ Database Architecture & Normalization Blueprint

To enforce strict relational integrity while preserving the high-leverage data fluidity required by travel designers, the data ecosystem is normalized into three distinct conceptual layers:

### 1. The Permanent Reference Layer (Master Dictionaries)
* **`countries`**: Serves as the global geographic anchor for destination research toolkits, storing static nation-level metadata.
* **`neighborhoods`**: Maps specific sectors (e.g., `Cape Town`, `Victoria Falls`) to their respective countries, eliminating string duplication and typo fragmentation.
* **`master_properties`**: A comprehensive master directory of reusable supplier, lodge, and activity provider records.

### 2. The Project Identity Layer
* **`trips`**: Serves as the structural "folder" anchoring all data for a specific client booking, tracking high-level parameters like client names and target travel windows.

### 3. The Workspace & Confirmation Ledger Layer
* **`itinerary_proposals`**: A highly fluid, transactional sandbox table where designers drop property records, map dynamic multi-currency quotes (ZAR/USD), manage sequence controls, and track custom client updates.
* **`itinerary_items`**: An immutable snapshot ledger that captures approved proposal lines upon confirmation to establish a permanent financial paper trail for accounting.

### Architectural Entity-Relationship Diagram (ERD)

========================================================================================
SYSTEM SCHEMATIC BLUEPRINT
[ LAYER 1: PERMANENT REFERENCE REPOSITORIES ]
+-----------------------+              +----------------------------+
| countries             |              | master_tags                |
+-----------------------+              +----------------------------+
| id (PK)               |              | id (PK)                    |
| country_name          |              | tag_label (e.g., 'Flight') |
+-----------------------+              +----------------------------+
	        |                                        |
	        v                                        |
+-----------------------+                          |
| neighborhoods         |                          |
+-----------------------+                          |
| id (PK)               |                          |
| neighborhood_name     |                          |
| country_id (FK)       |                          |
+-----------------------+                          |
	        |                                        |
	        v                                        v
+-------------------------------------------------------------------+
| master_properties                                                 |
+-------------------------------------------------------------------+
| id (PK)                                                           |
| property_name                                                     |
| neighborhood_id (FK)                                              |
+-------------------------------------------------------------------+
                             |
                             v
[ LAYER 2: THE TRIP FOLDER ]             [ LAYER 3: WORKSPACE SANDBOXES ]
+-----------------------+              +-----------------------------------+
| trips                 |              | itinerary_proposals (WIP Sandbox) |
+-----------------------+              +-----------------------------------+
| id (PK)               | <----------- | id (PK)                           |
| client_name           |              | trip_id (FK)                      |
| month_of_trip         |              | currency (ZAR / USD)              |
| sort_order (Integer)  |              | sort_order (Integer)              |
| invoice_number        |              | is_sector_header (Boolean)        |
+-----------------------+              | is_client_arranged (Boolean)      |
|                                      |                                   |
|                                      | -- 10-Column Spreadsheet Fields --|
|                                      | supplier_name (Text Safetynet)    |
|                                      | dates_text (Text Safetynet)       |
|                                      | total_cost / deposit_amount       |
|                                      | charged_client / net_profit       |
+------------------------>             +-----------------------------------+
|                                                    | (On Confirmation)
v                                                    v
+-------------------------------------------------------------------+
| itinerary_items (Final Confirmed Immutable Ledger)                |
+-------------------------------------------------------------------+
| id (PK)            | trip_id (FK)          | proposal_source_id   |
| currency           | sort_order            | financial_snapshot   |
+-------------------------------------------------------------------+


---

## ⚡ Key Engineering Highlights & Practical Solutions

### 📊 1. "Formula-Live" Programmatic Spreadsheet Generation
A core system requirement was keeping financial math reactive once extracted from the database. Instead of generating flat, static cell outputs, the export pipeline runs a browser-side binary stream using `exceljs` to inject live Microsoft Excel string formulas (such as `=C7-D7` or `=I7/C7`) directly into the cell buffers. When the travel designer interacts with the sheet offline, the calculations behave as a native workbook, adapting instantaneously to manual client adjustments or margin re-calculations.

### 🔄 2. Bidirectional Ingestion Loop via Hidden UUID Anchoring
To close the loop and allow offline spreadsheet edits to safely sync back into the web database, the export engine embeds the corresponding Supabase `UUID` identifier of every active line item into a hidden tracking column within the generated spreadsheet. 
Upon file upload, a custom extraction script parses the local file buffer and evaluates the identifier:
* **Match Found:** Executes an atomic database `upsert` block to cleanly mutate the record.
* **Blank Identifier Field:** Flags the entry as a newly appended itinerary component, generating a fresh relational database reference on the fly.

### 🛠️ 3. Structural Layout Controls & Text-Type Safety Valves
* **Visual Chunking Rows:** Travel itineraries require non-financial text separators (e.g., `== RWANDA SECTOR ==`) to group phases of a safari. The system models this through a boolean `is_sector_header` attribute. When flagged, it triggers a UI style rule and forces the spreadsheet export compiler to lock numeric columns on that line to zero, protecting summaries from calculation errors.
* **Flexible Input Typing:** Real-world travel logistics frequently break standard date types with inputs like `"23 May - 02 June"` or `"TBD - Mid Afternoon Flight"`. Storing transactional dates as text parameters backed by integer-based `sort_order` sequences ensures absolute presentation stability without database exceptions.

---

## 🔧 Conceptual Trade-Off Assessment

### Editable Web-Grids vs. File-Orchestration
During the discovery phase, building a complex, editable data grid using reactive state arrays directly on the frontend was heavily evaluated. While a native browser table provides real-time database syncing, it ultimately introduces severe performance bottlenecks when handling dozens of multi-currency loops, nested columns, and sudden text overrides. 

Delegating the heavy math tracking to local Excel engines while using PostgreSQL as the central source of truth significantly reduced frontend complexity, cut bundle sizes, and preserved 100% of the operational layout speed requested by the business stakeholders.

